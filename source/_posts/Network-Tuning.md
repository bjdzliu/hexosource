---
title: Network Performance Tuning
date: 2021-04-15 13:56:27
tags: Performance

---
### 数据如何从网卡到应用

```
NIC                    Kernel            Protocols           To App
+---+----------+       +---------+       +-----------+       +--------+
| P |  (    )  | ----> | SoftIRQ | ----> | Ethernet  | ----> | Socket |
| H | ( ring ) |       +---------+       |           |       | Buffer |
| Y |  (    )  |       |         |       | IP / ICMP |       +--------+
+---+----------+       | Backlog |       |           |
                       | (maybe) |       | TCP / UDP |                 
                       |         |       | SCTP      |                 
                       +---------+       +-----------+
```
- 数据进入 Ring Buffer
- NIC硬中断，停止正在运行的进程，上下文切换
- kernel 运行中断处理程序以关闭中断
- kernel 通过 SoftIRQ 来完成实际的接收工作
- SoftIRQ 线程获取数据，通过网卡提供的协议处理程序
- 数据放入到socket缓冲区
- SoftIRQ 线程继续轮询知道数据取完

### 网络性能瓶颈
#### 频繁中断
以腾讯云的一个vm为例
网卡虚拟化技术，使用的virtio。所以，分析vm的性能，了解virtio的方案是必要的。
以下显示CPU0在处理网卡的中断数量。
多个CPU，中断是否均匀

```
ubuntu@VM-0-5-ubuntu:~$ cat /proc/interrupts
           CPU0
  1:          9   IO-APIC   1-edge      i8042
  4:         90   IO-APIC   4-edge      ttyS0
  6:          3   IO-APIC   6-edge      floppy
  8:          0   IO-APIC   8-edge      rtc0
  9:          0   IO-APIC   9-fasteoi   acpi
 11:          0   IO-APIC  11-fasteoi   virtio2, uhci_hcd:usb1
 12:         15   IO-APIC  12-edge      i8042
 14:    4180743   IO-APIC  14-edge      ata_piix
 15:          0   IO-APIC  15-edge      ata_piix
 24:          0   PCI-MSI 98304-edge      virtio1-config
 25:   16000498   PCI-MSI 98305-edge      virtio1-req.0
 26:          0   PCI-MSI 81920-edge      virtio0-config
 27:   22148079   PCI-MSI 81921-edge      virtio0-input.0
 28:        359   PCI-MSI 81922-edge      virtio0-output.0

```

#### 合适的eth0队列
每个队列，绑定到不同的cpu上。
```
# ethtool -l eth0
Channel parameters for eth0:
Pre-set maximums:
RX:        0
TX:        0
Other:        0
Combined:    16  表示最多支持16个队列
Current hardware settings:
RX:        0
TX:        0
Other:        0
Combined:    8  表示当前是8个队列 rx-0 到 rx-7；tx-0 到tx-7

```
查看eth0的队列
```
ls /sys/class/net/eth0/queues/
rx-0 rx-1 rx-2 rx-3 rx-4 rx-5 rx-6 rx-7 tx-0 tx-1 tx-2 tx-3 tx-4 tx-5 tx-6 tx-7
```

#### 设置Ring Buffer
```
# ethtool --show-ring eth0
    Ring parameters for eth0:
    Pre-set maximums:
    RX:             4096
    TX:             4096
    Current hardware settings:
    RX:             256
    TX:             256
```
- data先放在NIC硬件中，kernel从中读取
- 最大值4096

#### offload 的介入
```
$ ethtool --show-features eth0|grep offload
tcp-segmentation-offload: on
udp-fragmentation-offload: off
generic-segmentation-offload: on
generic-receive-offload: on
large-receive-offload: off [fixed]
```

默认网卡上启动了offload，它的作用如下：
如果app发送一个7,300 Bytes的数据，它不会被在TCP层分成5个segment。而是作为一个整体，送到网卡上。
网卡将7,300 Bytes分成5个segment。


#### 检查在队列中的数据
```
$ cat /proc/net/softnet_stat
    7dab8f00 000004f6 000007a1 ...
    7dce863c 0000001d 0000079b ...
```

第一列-表示该cpu收到的包个数；
第二列-表示因softnet_data的输入队列满而丢弃的数据包个数（input_pkt_queue，队列长度最大值可通过/proc/sys/net/core/netdev_max_backlog调整）；
第三列-表示软中断一次取走netdev_budget个数据包，或取数据包时间超过2ms的次数；


修改netdev_max_backlog方式：
sysctl -a|grep netdev_max_backlog
net.core.netdev_max_backlog = 1000


#### TCP层控制
```
#netstat -s|egrep 'pruned|collapsed'

# netstat -s
    121093641 segments retransmited
    120 packets pruned from receive queue because of socket buffer overrun
    685394 packets collapsed in receive queue due to low socket buffer
    1804 times recovered from packet loss due to fast retransmit
    54444 congestion windows fully recovered
    45760872 fast retransmits
    23208494 retransmits in slow start
```
pruned: 由于套接字缓冲区溢出而从接收队列中删除的数据包  
collapsed: Socket buffer满导致包无法放入

- 调整socket tcp 缓冲区
三列分别是：min default max
```
/etc/sysctl.conf
net.ipv4.tcp_rmem = 4096 262144 16777216
net.ipv4.tcp_wmem = 4096 262144 16777216
```

- 调整socket 非TCP 缓冲区
tcp的缓冲区值，受限于这里的max
```
    net.core.rmem_max = 16777216
    net.core.wmem_max = 16777216
    net.core.rmem_default = 262144
    net.core.wmem_default = 262144
```

- 讨论是否打开TCP Timestamps

默认打开：
```
cat /proc/sys/net/ipv4/tcp_timestamps
1
```
如果是NAT的场景，不要使用这个值。原因是：如果clientB也是通过lvs服务器访问到server1，那么因为之前clientA已经访问到server1，并且lvs配置了tcp_tw_recycle=1；tcp_timestamps=1；那么lvs作为 “源ip：源port” 如果重复，就会被server1抛弃。显现是有syn，没有ack。


tcp_tw_recycle 再kernel 4.x版本后已经废弃；  
tcp_tw_recycle 表明尽快的回收处于 time_wait 状态的连接，不用等两个 MSL 就关闭连接。
               缺点是：会拒绝所有比这个客户端时间戳更靠前的网络包。在开启

- SACK
SACK是接收方用来向发送方通知已经接收到哪些序列号段的一种机制，这样发送方在重传时就只需要重传接收方真正未收到的部分即可。

```
root@VM-0-5-ubuntu:/proc# sysctl -a|grep tcp_sack
net.ipv4.tcp_sack = 1
```


#### 应用层控制
现象：
```
[root@VM_0_15_centos ~]# netstat -s | grep -i listen    
62678 times the listen queue of a socket overflowed    ---- >应用处理全连接队列(accept queue)过慢导致；
65640 SYNs to LISTEN sockets dropped  ----> 半链接满了
```
可调的位置：
针对accept queue，调整
修改全连接
/proc/sys/net/core/somaxconn
默认128

或者，调整应用层的backlog，比如：
listen 443 ssl backlog=20480;

全连接和半连接的关系：
![全连接和半连接的关系](https://bjdzliu.oss-cn-beijing.aliyuncs.com/hexo_images/sync-queue.png)
