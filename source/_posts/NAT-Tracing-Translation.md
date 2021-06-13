---
title: NAT-Tracing(Translation)
date: 2021-06-12 21:50:46
tags:
- k8s
- network
categories:
- Technical Notes
---

### 介绍
NAT是可以将Container和VM的内部地址暴露在公网中。传入的连接请求的目的地址被重写成另一个地址。然后，Packet经路由送到container和VM中。这也被用在LB的技术中。  
NAT失效即意味请求失败。同样的情况也发生在：暴露了错误的服务，container中断连接、连接超时等。
定位这类问题的方法：判断请求和预期得到的地址是否一致。

### 连接跟踪(Connection tracking）
NAT技术不仅仅是改变IP、改变port。例如，当地址 X 映射到 Y 时，不需要运维人员添加相应Rule。被称作“conntrack”的过滤系统，它会识别回复给现有连接的 Packet，这意味着：每个连接有自己的NAT状态，反向翻译是自动完成的。

### 规则集跟踪评估
实用程序 nftables 允许检查 Packet 如何评估，以及可以匹配规则集中的哪些规则。为了使用这个特性，“跟踪规则”被插入到一个合适的位置。这些规则选择应该跟踪的 Packet。假设 IP 地址为 c 的主机试图访问服务：service:port。我们想知道哪些 NAT 被选中，哪些规则被检查，以及数据包是否被丢弃在某个地方。

因为我们正在处理传入的连接，所以向 Prerouting hook点添加一个规则。Prerouting 意味着内核还没有决定把数据包发送到哪里。在Prerouting上更改其目的地址通常有两种结果，数据包被转发，或者由本机处理。

### Initial setup
```
# nft 'add table inet trace_debug'
# nft 'add chain inet trace_debug trace_pre { type filter hook prerouting priority -200000; }'
# nft "insert rule inet trace_debug trace_pre ip saddr $C ip daddr $S tcp dport $P tcp flags syn limit rate 1/second meta nftrace set 1"
```

第一个命令：新建一个table。基于这个table，做跟踪和调试，删除也很简单。“nft delete table inet trace_debug”


第二个命令：路由前创建一个钩子，在Prerouting时。首先执行，即在NAT和conntrack跟踪前。

第三个命令：(bjdzliu：不需要跟踪那么多包，一秒内或一分钟内取一个包)  
“meta nftrace set 1”。这样就可以跟踪与规则匹配的所有数据包的事件。尽可能设计一个好的信噪比。考虑添加速率限制，以便将跟踪事件的数量保持在可管理的水平。每秒或每分钟限制一个包是一个很好的选择。所提供的示例跟踪来自主机 $c 的所有 syn 和 syn/ack 数据包，并且跟踪到目标主机 $s 上的目的端口 $p。Limit 子句防止事件泛滥。在大多数情况下，追踪一个数据包就足够了。

类似用Iptables命令跟踪
```
# iptables -t raw -I PREROUTING -s $C -d $S -p tcp --tcp-flags SYN SYN  --dport $P  -m limit --limit 1/s -j TRACE
```

### 获取跟踪事件

```
# nft monitor trace
```
打印出所有收到的 Packet 及 和匹配这个 Packet 的Rule

```
trace id f0f627 ip raw prerouting  packet: iif "veth0" ether saddr ..

```
我们将在下一节中更详细地研究这个打印出来的记录。如果您使用 iptables，首先通过“ iptables-version”命令检查已安装的版本。例子:

```
# iptables --version
iptables v1.8.5 (legacy)
```

(legacy)意味着跟踪事件被记录到内核环缓冲区。你需要检查 dmesg 或者 journalctl。调试输出缺少一些信息，但在概念上与新工具提供的信息相似。您需要检查记录的规则行号，并将其与您自己的活动 iptables 规则关联起来。如果输出显示(nf _ tables) ，您可以使用 xtables-monitor 工具:


```
# xtables-monitor --trace
```
如果该命令只显示版本，则还需要查看 dmesg/journalctl。Xtables-monitor 使用与 nft 监视器跟踪工具相同的内核接口。它们唯一的区别是，它将以 iptables 语法打印事件，而且如果混合使用 iptables-nft 和 nft，它将无法打印使用映射/集和其他仅使用 nftable 的特性的规则。


### 例子
模拟场景：
10.1.2.3是host，通过1222，转发到host中的container中。
问题：ssh连接container超时
```
ssh -p 1222 10.1.2.3

```

Debug：
登陆host，添加追踪：

```
nft "insert rule inet trace_debug trace_pre ip daddr 10.1.2.3 tcp dport 1222 tcp flags syn limit rate 6/minute meta nftrace set 1"
```

添加Rule后，启动nft追踪“nft monitor trace”，然后重试ssh命令。产生如下输出：

```
trace id 9c01f8 inet trace_debug trace_pre packet: iif "enp0" ether saddr .. ip saddr 10.2.1.2 ip daddr 10.1.2.3 ip protocol tcp tcp dport 1222 tcp flags == syn
trace id 9c01f8 inet trace_debug trace_pre rule ip daddr 10.2.1.2 tcp dport 1222 tcp flags syn limit rate 6/minute meta nftrace set 1 (verdict continue)
trace id 9c01f8 inet trace_debug trace_pre verdict continue
trace id 9c01f8 inet trace_debug trace_pre policy accept
trace id 9c01f8 inet nat prerouting packet: iif "enp0" ether saddr .. ip saddr 10.2.1.2 ip daddr 10.1.2.3 ip protocol tcp  tcp dport 1222 tcp flags == syn
trace id 9c01f8 inet nat prerouting rule ip daddr 10.1.2.3  tcp dport 1222 dnat ip to 192.168.70.10:22 (verdict accept)
trace id 9c01f8 inet filter forward packet: iif "enp0" oif "veth21" ether saddr .. ip daddr 192.168.70.10 .. tcp dport 22 tcp flags == syn tcp window 29200
trace id 9c01f8 inet filter forward rule ct status dnat jump allowed_dnats (verdict jump allowed_dnats)
trace id 9c01f8 inet filter allowed_dnats rule drop (verdict drop)
trace id 20a4ef inet trace_debug trace_pre packet: iif "enp0" ether saddr .. ip saddr 10.2.1.2 ip daddr 10.1.2.3 ip protocol tcp tcp dport 1222 tcp flags == syn
```

### 逐行分析
生成的第一行是触发后续跟踪输出的 Packet id。尽管这与 nft 规则语法相同，但是它包含刚刚收到的数据包的头字段 Header。Packet是哪个网络接口接收的(这里是“enp0”)、Packet 的源和目的 mac 地址、源 ip 地址(是期望的源IP吗？)以及 tcp 源端口和目的端口。还有一个“trace id”。这个id告诉进入的数据包与规则匹配。  

第二行包含与数据包匹配的第一条规则:
```
trace id 9c01f8 inet trace_debug trace_pre rule ip daddr 10.2.1.2 tcp dport 1222 tcp flags syn limit rate 6/minute meta nftrace set 1 (verdict continue)
```

你看，上面一行，就是我们自己添加的Rule-A。这个Rule-A激活了跟踪Packet的行动！！如果有其他Rule-B在它之前，或者没有合适的Packet，那Rule-A不会生效。

第三、第四行：
accept Packet

第六行，匹配规则：
```
trace id 9c01f8 inet nat prerouting rule ip daddr 10.1.2.3  tcp dport 1222 dnat ip to 192.168.70.10:22 (verdict accept)
```

此规则识别到一个重写地址和端口。如果192.168.70.10确实是所需 VM/Container 的地址，到目前为止没有问题。如果它不是正确的 VM/Container 地址，那么这个地址要么是输错了，要么是 NAT 规则匹配错误。

### IP forwarding
Prerouting后，执行路由判断  
需要转发 Packet 到另一个 host，这个过程也被dump了出来。
```
trace id 9c01f8 inet filter forward packet: iif "enp0" oif "veth21" ether saddr .. ip daddr 192.168.70.10 .. tcp dport 22 tcp flags == syn tcp window 29200

```
我们看到出现了一个 output 的网络接口-veth21。trace id没变，所以仍然是之前packet，只是地址和端口已经更改。  
如果有与 “tcp dport 1222” 匹配的Rule，那么这些Rule将不再对该 paacket 产生影响。

如果没有output 的网络接口-- oif "veth21"，路由就会把packet转给local host。

```
trace id 9c01f8 inet filter forward rule ct status dnat jump allowed_dnats (verdict jump allowed_dnats)

```
上面这条：显示packet匹配到一个 chain -> allowed_dnats.

下面这条：显示了连接失败的来源，该规则无条件地丢弃数据包，因此不存在进一步的数据包日志输出:
```
trace id 9c01f8 inet filter allowed_dnats rule drop (verdict drop)

```
 (bjdzliu: filter表中的allowed_dnats拦截)

下面这条：trace id已经变了，输出行是另一个数据包的结果:

```
trace id 20a4ef inet trace_debug trace_pre packet: iif "enp0" ether saddr .. ip saddr 10.2.1.2 ip daddr 10.1.2.3 ip protocol tcp tcp dport 1222 tcp flags == syn

```
trace id 不同，但是 packet 具有相同的内容。这是一次重传尝试: 第一个数据包被丢弃，因此 TCP 重试。忽略其余的输出，它不包含新信息。是时候检查chain了。


###  规则调查
前面的部分发现 packet 被丢弃在 inet filter表 中一个名为 “allowed_dnats” 的链中。我们看一下：

```
# nft list chain inet filter allowed_dnats
table inet filter {
 chain allowed_dnats {
  meta nfproto ipv4 ip daddr . tcp dport @allow_in accept
  drop
   }
}
```
接受 @allow_in 集中的数据包，在跟踪日志中没看到这样的记录log。再次检查地址是否在@allow_set 中，命令如下:


```
# nft "get element inet filter allow_in { 192.168.70.10 . 22 }"
Error: Could not process rule: No such file or directory
```
和推测的一样，192 这个地址不在set中。

手工添加：

```
# nft "add element inet filter allow_in { 192.168.70.10 . 22 }"

```

再次检查
```
# nft "get element inet filter allow_in { 192.168.70.10 . 22 }"
table inet filter {
   set allow_in {
      type ipv4_addr . inet_service
      elements = { 192.168.70.10 . 22 }
   }
}
```

ssh工作ok：

```
trace id 497abf58 inet filter forward rule ct status dnat jump allowed_dnats (verdict jump allowed_dnats)

```

```
trace id 497abf58 inet filter allowed_dnats rule meta nfproto ipv4 ip daddr . tcp dport @allow_in accept (verdict accept)

```

```
trace id 497abf58 ip postrouting packet: iif "enp0" oif "veth21" ether .. trace id 497abf58 ip postrouting policy accept

```

### 总结
本文介绍了如何使用 nftables 跟踪机制检查数据包丢失和其他连接性问题的来源。
这是翻译的第一篇。作者还有后续第二篇。
