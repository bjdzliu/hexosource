---
title: NAT-Tracing-part2(Translation)
date: 2021-06-13 21:57:31
tags:
- k8s
- network
categories:
- Technical Notes
---
### 介绍
通过 iptables 或 nftables 配置的 NAT 构建在 netfilters 连接跟踪工具之上。 conntrack 命令用于检查和更改状态表。 它是“conntrack-tools”包的一部分。

### 跟踪状态表
连接跟踪子系统跟踪它所看到的所有数据包流。运行 “sudo conntrack-l” 查看其内容:
```
tcp 6 43184 ESTABLISHED src=192.168.2.5 dst=10.25.39.80 sport=5646 dport=443 src=10.25.39.80 dst=192.168.2.5 sport=443 dport=5646 [ASSURED] mark=0 use=1
tcp 6 26 SYN_SENT src=192.168.2.5 dst=192.168.2.10 sport=35684 dport=443 [UNREPLIED] src=192.168.2.10 dst=192.168.2.5 sport=443 dport=35684 mark=0 use=1
udp 17 29 src=192.168.8.1 dst=239.255.255.250 sport=48169 dport=1900 [UNREPLIED] src=239.255.255.250 dst=192.168.8.1 sport=1900 dport=48169 mark=0 use=1
```

其中第一列为该记录的协议，即TCP。

第二列为协议号，TCP协议号为6。

第三列是该条记录的剩余生存时间，该时间会持续减少直到有新的数据包到达来更新该记录。

第四列是协议过程中的状态，表明当前该记录处在TCP的SYN_SENT状态，即发出了syn包，等待应答。

之后便是维护当前连接数据包的源IP、目的IP、源端口、目的端口。

中括号内的是连接的状态，[UNREPLIED]表示只有单向有数据传输，没有应答。当收到回复，数据变为双向传输后，连接跟踪变为删除该标记。当出现[ASSURED]时，表明该记录在两个方向上都没有数据传输，存在该标记的记录在连接跟踪表满时不会被删除。

中括号后是该连接所期望收到的数据包的源IP、目的IP、源端口和目的端口号，可见和之前的前面的一条记录是完全相反的。

- 如果 NAT 规则匹配，比如 IP masquerade，这会记录在连接跟踪条目的 replay 部分，然后可以自动应用到同一流的所有未来数据包。

原始的请求，即第一个四元组永远不会更改: 它是发起者发送的内容。NAT 操作只会修改replay部分，即第二个四元组，因为接收方将看到这一点。Netfilter 不能控制发起者的状态，它只能在接收/转发数据包时影响数据包。当数据包没有映射到现有条目时，conntrack 可以为它添加一个新的状态条目。在 UDP 的情况下，这是自动发生的。在 TCP 连接的情况下，如果 TCP 数据包设置了 SYN 位，则配置为只添加新条目。  
Conntrack 不会影响之前已经创建好的数据流。


### Conntrack state table and NAT
可以过滤输出发生NAT转换的条目：
“sudo conntrack-l-p tcp-src-nat”

```
tcp 6 114 TIME_WAIT src=10.0.0.10 dst=10.8.2.12 sport=5536 dport=80 src=10.8.2.12 dst=192.168.1.2 sport=80 dport=5536 [ASSURED]
```

该条目表明从本机发起的，从 10.0.0.10:5536 到 10.8.2.12:80 的请求。
replay 的四元组并不是请求的四元组。
期望：目的主机 10.8.2.12 发的replay包到，其目的地址由10.0.0.10改成了192.168.1.2  
这是因为：每当10.0.0.10发送另一个数据包时，使用该条目的路由器将源地址替换为192.168.1.2，即发生SNAT。那么从 10.8.2.12 回的地址，就是SNAT后的地址。
当10.8.2.12发送回复时，目的地更改为10.0.0.10。

```
inet nat postrouting meta oifname "veth0" masquerade

```
其他类型的 NAT 规则类似，比如 “dnat to” 或者 “redirect to”。

### Conntrack extensions
```
sudo sysctl net.netfilter.nf_conntrack_acct=1
sudo conntrack -L
记录时间戳：
sudo sysctl net.netfilter.nf_conntrack_timestamp=1
```

### 插入或更新条目
```
sudo conntrack -I -s 192.168.7.10 -d 10.1.1.1 --protonum 17 --timeout 120 --sport 12345 --dport 80
```

```
conntrack -U -m 42 -p tcp
```


### 删除条目
```
sudo conntrack -D -p udp  --src 10.0.12.4 --dst 10.0.0.1 --sport 1234 --dport 53
```

### 错误统计
```
# sudo conntrack -S
cpu=0 found=0 invalid=130 insert=0 insert_failed=0 drop=0 early_drop=0 error=0 search_restart=10
cpu=1 found=0 invalid=0 insert=0 insert_failed=0 drop=0 early_drop=0 error=0 search_restart=0
cpu=2 found=0 invalid=0 insert=0 insert_failed=0 drop=0 early_drop=0 error=0 search_restart=1
cpu=3 found=0 invalid=0 insert=0 insert_failed=0 drop=0 early_drop=0 error=0 search_restart=0
```


```
nf_ct_proto_6: invalid tcp flag combination SRC=10.0.2.1 DST=10.0.96.7 LEN=1040 TOS=0x00 PREC=0x00 TTL=255 ID=0 PROTO=TCP SPT=5723 DPT=443 SEQ=1 ACK=0
```

### 总结
[原文](https://fedoramagazine.org/network-address-translation-part-2-the-conntrack-tool/)
