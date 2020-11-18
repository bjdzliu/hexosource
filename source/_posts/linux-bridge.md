---
title: linux_bridge
date: 2017-04-20 16:27:57
tags:
- linux
categories:
- Technical Notes
---
### Linux Bridge and Virtual Networking
学习linux网卡知识时遇到的一篇文章，非常基础。  

[原文](https://cloudbuilder.in/blogs/2013/12/02/linux-bridge-virtual-networking/)
*看到13年的文章，有点感慨学习的有些晚了*   
Linux Bridge 和 虚拟网络      
软件定义网络 (SDN) 当前正席卷网络产业。SDN 的关键功能之一就是网络虚拟化。sdn和网络虚拟化在当前已经是潮流，但他们并不是什么新的技术，Linux bridge才是这个领域的最早实践者。  

##### Linux Bridge的概要    
网络虚拟化就是实现在server/hypervisor内部的虚拟交换机(switch)功能。虽然叫做bridge(桥)，但他实在是一个虚拟交换机，并配合KVM/QEMU的虚拟化。Linux Bridge是linux的内核功能，用工具brctl管理。    


##### 简单使用  
看一个基本的使用案例，你想在kvm上创建一个VM，除此之外，vm配置一块NIC网卡，如果在vm里想通过nic网卡上网，需要nic和宿主机网卡做关联。这个关联通过Linux bridge 实现。实现的图片如下：

![Simple Use Case for Linux Bridge](https://bjdzliu.oss-cn-beijing.aliyuncs.com/hexo_images/linux_network/Linux-Bridge-Simple-UseCase.png)

上面图片解释：一台运行着kvm虚拟化环境的ubuntu笔记本。笔记本通过wlan0连接。为了说明Linux bridge的功能，我准备创建一个VM，将VM的关联到我的笔记本的有线网卡eth0上。

##### 步骤：
1. 用brctl创建Linux bridge。brctl命令解释
`# sudo brctl addbr kvmbr0`
2. 将物理机的eth0网卡关联到此Bridge，注意：此步骤前确保物理网卡eth0上没有任何IP地址。
`# sudo brctl addif kvmbr0 eth0`
执行完成后，网络状态如下：
现在kvmbr0 bridge只有一个网络接口(eth0)
![Linux Bridge Interface Config Sample](https://bjdzliu.oss-cn-beijing.aliyuncs.com/hexo_images/linux_network/Linux-Bridge-Interface-Config.png)


3. 下一步创建VM并让这个VM使用刚才创建的Linux bridge。这篇blog通过使用“Virtual Machine Manager” 的GUI界面进行配置。  
![Associate Linux Bridge to a VM](https://bjdzliu.oss-cn-beijing.aliyuncs.com/hexo_images/linux_network/Linux-Bridge-Virt-Manager.png)  

一旦VM被创建并启动，此VM就有一个外部的网络连接。  

##### 研究网络端口  
Brctl show命令显示在kvmbr0 bridge上有另外一个网络接口vnet0。**libvirt创建一块虚拟网卡** **_veth0_**, 见下图。这块虚拟网卡也被叫做 tap interface. 通过ps KVM/QEMU 可以看到启动vm时，指定的网络设备为 **_tap interface_**。

![Linux Bridge with Virtual Interface](https://bjdzliu.oss-cn-beijing.aliyuncs.com/hexo_images/linux_network/Screenshot-2013-12-02-22_43_28.png)

就像你用网线连接物理机和交换机一样，VM的虚拟NIC和Bridge上的tap interface相连。下图展示了vm的NIC和linux bridge 的tap设备的关系。  
1）注意eth0和vnet0的mac地址相似。  
2）观察两者的数据发送和接收。因为存在1:1的关系， VM NIC的TX byte和vnet0的RX byte基本一致。  
3）vm中的Virtual NIC 配置了IP和路由，这是通过宿主机上的DHCP服务提供的。这也说明此Virtual NIC已经有了外部网络连接。  

![VM NIC to Tap Interface relationship](https://bjdzliu.oss-cn-beijing.aliyuncs.com/hexo_images/linux_network/Screenshot-2013-12-02-22_46_24.png)


#### 总结：
我们创建了一个Linux网桥，并添加了主机的物理NIC接口。  
然后在创建虚拟机时，我们指定了用于虚拟网络的Linux网桥。  
虚拟机管理器（libvirt GUI）做了一些幕后工作，将虚拟网卡连接到Linux网桥，然后依次连接到物理网卡。    
然后，我们观察了VM的虚拟NIC如何与主机上的虚拟分接口相关联。 Tap接口如何添加到Linux网桥。  
这表明流量将从虚拟机的虚拟网卡流向vnet0分接口，然后流入Linux网桥（虚拟交换机），该桥接器将在主机上的另一个虚拟交换机接口（eth0）上发送。  
