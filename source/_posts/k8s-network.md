---
title: k8s_network
date: 2017-05-08 16:19:30
tags:
- k8s
categories:
- Technical Notes
---
k8s网络设计三个层面的网络知识：
- docker network
- Third party network
- k8s network


#### Third party network
以Calico为例：  
-  Glossary:  
CNI : Container Network Interface  
CNM : container network model

The direct routes are set up by a Calico agent named **_Felix_**.    
Calico also programs *iptables* on each host,This programming is bookended'.  
Endpoints are the TAPs, veths or other interfaces.  

kubelet启动参数  
>–cni-bin-dir string The full path of the directory in which to search for CNI plugin binaries. Default: /opt/cni/bin   
–cni-conf-dir string The full path of the directory in which to search for CNI config files. Default: /etc/cni/net.d

kubelet运行命令
```
/usr/bin/kubelet –kubeconfig=/etc/kubernetes/kubelet.conf \
–require-kubeconfig=true\
–pod-manifest-path=/etc/kubernetes/manifests \
–allow-privileged=true \
–network-plugin=cni \
–cni-conf-dir=/etc/cni/net.d \
–cni-bin-dir=/opt/cni/bin \
–cluster-dns=10.96.0.10 \
–cluster-domain=cluster.local\
–authorization-mode=Webhook \
–client-ca-file=/etc/kubernetes/pki/ca.crt
```

[Calico的yaml定义](https://docs.projectcalico.org/v2.0/getting-started/kubernetes/installation/hosted/calico.yaml)


–cni-conf-dir=/etc/cni/net.d —- >对应--> YAML: Hostpath: /etc/cni/net.d ( 映射calico/cni:v1.4.2 container目录 /host/etc/cni/net.d )  

–cni-bin-dir=/opt/cni/bin —- >对应--> YAML: HostPath: /opt/cni/bin ( 映射calico/cni:v1.4.2 container目录 mountPath: /host/opt/cni/bin )

YAML文件中： image quay.io/calico/node:v0.22.0 使用配置configmap的配置参数 etcd_endpoints and enable_bgp

YAML文件中： image calico/cni:v1.4.2 使用配置configmap的配置参数 etcd_endpoints and cni_network_config  
这个image calico/cni:v1.4.2 的启动命令：  “/install-cni.sh”  
它的作用：将二进制文件和配置文件映射到宿主机上, 这些文件和目录 供 kubelet 启动 pod network 时使用. 对这些文件的需求, 满足[这里](https://github.com/containernetworking/cni/blob/master/SPEC.md#network-configuration)。


calico node container 和 cni container 在一个pod里。  
- node container 的作用:  
 Runs calico/node container on each Kubernetes node. This container programs network policy and routes on each host

- cni container的作用:  
  This container installs the Calico CNI binaries and CNI network config file on each node.

##### Test in laptop
使用代理
or
把下面几行内容加入到/etc/hosts文件中
61.91.161.217 gcr.io  
61.91.161.217 www.gcr.io  
61.91.161.217 packages.cloud.google.com  
最新可用的Google hosts文件可在 [这里](https://github.com/racaljk/hosts  ) 获取

1 使用kubeadm安装,Follow [this](https://kubernetes.io/docs/getting-started-guides/kubeadm/)  
2 安装 pod network

```
sudo cp /etc/kubernetes/admin.conf $HOME/
sudo chown $(id -u):$(id -g) $HOME/admin.conf
export KUBECONFIG=$HOME/admin.conf
kubectl –kubeconfig=/etc/kubernetes/kubelet.conf describe nodes
```
设置为单点:  
`kubectl –kubeconfig=/etc/kubernetes/kubelet.conf taint nodes –all node-role.kubernetes.io/master-`

补充:  
Annotation则是用户任意定义的“附加”信息，以便于外部工具进行查找。
用annotation来记录的信息包括：
build信息、release信息、docker镜像信息等，如时间戳、release id号、PR号、镜像hash值、docker Controller地址等

**_cluster-cidr_**  参数，用来通过linux bridge模式设置kubernetes集群中POD的IP地址范围。
>kube-proxy : The CIDR range of pods in the cluster. It is used to bridge traffic coming from outside of the cluster. If not provided, no off-cluster bridging will be performed.
kube-conroller: string CIDR Range for Pods in cluster.

**_Controller Manager_** 在启动时如果设置了 -cluster-cidr 参数，那么为每个没有设置spec.podCIDR的node生成一个CIDR地址，并用该 CIDR 设置节点的 spec.PodCIDR 属性，这样的目的是防止不同节点的 CIDR 地址发生冲突。

安装dashboard
```
kubectl –kubeconfig=/etc/kubernetes/kubelet.conf –namespace=default create -f https://rawgit.com/kubernetes/dashboard/master/src/deploy/kubernetes-dashboard.yaml
```

kubeadm 安装方式:
```
kubectl –kubeconfig=/etc/kubernetes/kubelet.conf get services
kubectl –kubeconfig=/etc/kubernetes/kubelet.conf apply -f http://docs.projectcalico.org/v2.1/getting-started/kubernetes/installation/hosted/kubeadm/1.6/calico.yaml
```

- 查看搭建后的IP
pod的IP地址为 192.168.219.194  
docker0的地址为 172.17.0.1  
容器内的IP:
>4: eth0@if9: mtu 1500 qdisc noqueue state UP
link/ether fa:2c:39:ba:57:c9 brd ff:ff:ff:ff:ff:ff
inet 192.168.219.193/32 scope global eth0
valid_lft forever preferred_lft forever
inet6 fe80::f82c:39ff:feba:57c9/64 scope link
valid_lft forever preferred_lft forever

```
root@ubuntux230:/etc/cni/net.d# ls -ltr
total 8
-rw-rw-r– 1 root root 1289 May 7 02:40 10-calico.conf
-rw-r–r– 1 root root 273 May 7 02:40 calico-kubeconfig

/opt/cni/bin also exists.
```


- 察看宿主机的docker network

root@ubuntux230:/opt/cni/bin# docker network inspect none
“Containers”: {

“**_a869ccc_** 2a59f990df3db30546dbcef6e7ac7210b55c2eb392dc8c44c0c525328”: {
“Name”: “k8s_POD_kubernetes-dashboard

“**_d4dc5c0_** e5778c360a85d3d96436cdb54ecf7f0579f9b4d4bf9aad1d39a821853”: {
“Name”: “k8s_POD_kube-dns-3913472980-gtkd3_kube-system_83904260-32ef-11e7-84be-3c970ee8055c_0”,

**_网络是由 pause-amd64:3.0 容器构建的。_**

root@ubuntux230:/opt/cni/bin# docker ps -a|grep **_a869ccc_**
>a869ccc2a59f gcr.io/google_containers/pause-amd64:3.0 “/pause” About an hour ago Up About an hour k8s_POD_kubernetes-dashboard-2457468166-5z067_kube-system_6054a874-32f0-11e7-84be-3c970ee8055c_0

root@ubuntux230:/opt/cni/bin# docker ps -a|grep **_d4dc5c0_**
>d4dc5c0e5778 gcr.io/google_containers/pause-amd64:3.0 “/pause” About an hour ago Up About an hour k8s_POD_kube-dns-3913472980-gtkd3_kube-system_83904260-32ef-11e7-84be-3c970ee8055c_0

查看使用 host网络 的 POD：  
`docker network inspect host`

宿主机网络  
>8: tunl0@NONE: mtu 1440 qdisc noqueue state UNKNOWN group default qlen 1
link/ipip 0.0.0.0 brd 0.0.0.0  
inet 192.168.219.192/32 scope global tunl0  
valid_lft forever preferred_lft forever  
9: califdbcd38298d@if4: mtu 1500 qdisc noqueue state UP group default
link/ether 66:65:35:99:aa:1f brd ff:ff:ff:ff:ff:ff link-netnsid 0  
inet6 fe80::6465:35ff:fe99:aa1f/64 scope link  
valid_lft forever preferred_lft forever  
10: caliebbe646ea97@if4: mtu 1500 qdisc noqueue state UP group default
link/ether ee:24:72:76:64:7b brd ff:ff:ff:ff:ff:ff link-netnsid 1  
inet6 fe80::ec24:72ff:fe76:647b/64 scope link  
valid_lft forever preferred_lft forever  

宿主机路由:
>root@ubuntux230:/opt/cni/bin# ip route
default via 9.115.114.1 dev kvmbr0  
9.115.114.0/24 dev kvmbr0 proto kernel scope link src 9.115.114.219  
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown  
blackhole 192.168.219.192/26 proto bird  
192.168.219.193 dev califdbcd38298d scope link 其中一个pod  
192.168.219.194 dev caliebbe646ea97 scope link  

查看kube-dns pod网络
kube-dns IP： 192.168.219.193
root@ubuntux230:/opt/cni/bin# docker exec -ti 6677c4f4bb7b ip route —- 路由 kube-dns的container
>default via 169.254.1.1 dev eth0  
169.254.1.1 dev eth0  

169.254.1.1的来源[解释](http://docs.projectcalico.org/v2.0/usage/troubleshooting/faq#why-does-my-container-have-a-route-to-16925411)


如果使用flannel架构，查看路由如下：  
root@ntc232:~# docker exec -ti e40335026640 ip route
>default via 33.89.0.1 dev eth0
33.89.0.0/24 dev eth0 proto kernel scope link src 33.89.0.6

如果使用flannel架构，查看搭建后的IP如下
root@ubuntux230:/opt/cni/bin# docker exec -ti 6677c4f4bb7b ip addr —- IP kube-dns的container
>1: lo: mtu 65536 qdisc noqueue state UNKNOWN qlen 1
link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
inet 127.0.0.1/8 scope host lo
valid_lft forever preferred_lft forever
inet6 ::1/128 scope host
valid_lft forever preferred_lft forever
2: tunl0@NONE: mtu 1480 qdisc noop state DOWN qlen 1
link/ipip 0.0.0.0 brd 0.0.0.0
4: eth0@if9: mtu 1500 qdisc noqueue state UP
link/ether fa:2c:39:ba:57:c9 brd ff:ff:ff:ff:ff:ff
inet 192.168.219.193/32 scope global eth0
valid_lft forever preferred_lft forever
inet6 fe80::f82c:39ff:feba:57c9/64 scope link


**参考链接**  
[kubernetes Networking and Network Policy](https://kubernetes.io/docs/admin/addons/)
[kubernetes Networking plugin](https://kubernetes.io/docs/concepts/cluster-administration/network-plugins/)  
[projectcalico](http://docs.projectcalico.org/v2.0/getting-started/kubernetes/installation/hosted/kubeadm/)  
[calico+docker](https://www.cnblogs.com/zhenyuyaodidiao/p/5322782.html)
[calico+docker](https://github.com/containernetworking/cni/blob/master/SPEC.md#network-configuration)  
使用CNI的过程 follow [This](https://github.com/containernetworking/cni)
