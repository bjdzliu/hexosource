---
title: Install Ceph
date: 2017-11-11 09:19:41
tags:
- ceph
categories:
- Technical Notes
---
### INSTALL CEPH ON A SINGLE NODE(BARE METAL)
此文是在linuxone上创建ceph存储的例子。

同amd64架构。脱胎于ceph的官方文档。

如果是linuxone的机器，请设置/etc/hosts文件如下
YOUR.HOST.PUBLIC.IP hostname

更改iptables的规则，添加一条本机到本机的规则：  
`iptables -A INPUT -s YOUR.HOST.PUBLIC.IP -d YOUR.HOST.PUBLIC.IP -j ACCEPT`

更新安装  
`apt-get update && apt-get -y install ceph-deploy`

设置ssh本地无密码登陆：  
```
ssh-keygen
cat /root/.ssh/id_rsa.pub >> /root/.ssh/authorized_keys
```

```
mkdir my-cluster && cd my-cluster

ceph-deploy new hostname

echo “osd pool default size = 2” >> ceph.conf
echo “osd crush chooseleaf type = 0” >>ceph.conf
echo “osd journal size = 512” >>ceph.conf

echo “osd pool default pg num = 75” >>ceph.conf
echo “osd pool default pgp num = 75” >>ceph.conf

echo “osd_max_object_name_len = 256” >>ceph.conf
echo “osd_max_object_namespace_len = 64” >>ceph.conf

echo “filestore xattr use omap = true” >>ceph.conf
```
参考 [offical doc](http://docs.ceph.com/docs/hammer/rados/configuration/filesystem-recommendations/) ext4文件系统建议

```
ceph-deploy install hostname

apt-get install ceph-fs-common

ceph-deploy mon create-initial

mkdir /osd1

mkdir /osd2

mkdir /osd3

chown ceph:ceph /osd1
chown ceph:ceph /osd2
chown ceph:ceph /osd3

ceph-deploy osd prepare ceph01:/osd1 ceph01:/osd2 ceph01:/osd3

ceph-deploy admin ceph01

ceph-deploy osd activate ceph01:/osd1 ceph01:/osd2 ceph01:/osd3

ceph health

ceph -s
```

*检查cepth*
```
#ceph status
pgs is less than XXX
```

*计算PG*

>Total PGs = ((Total_number_of_OSD * 100) / max_replication_count) / pool_count
ceph osd pool set rbd pg_num 150
ceph osd pool set rbd pgp_num 150

*减少pg_num*
>Clear everything and reinstall ceph  
`ceph-deploy purge ceph01`

*创建Cepth FS：*
- 创建元数据服务器  
    `ceph-deploy mds create ceph01`
- 创建两个存储池  

```
ceph osd pool create cephfs_data 64
ceph osd pool create cephfs_metadata 64

ceph fs new cephshare cephfs_metadata cephfs_data

mount -t ceph -o name=admin,secretfile=/etc/ceph/admin.secret {MON’s IP}:/ /mnt/mycephfs

or

mount -t ceph -o name=admin,secret=AQDVJghZd56vHxAAOqREEflJFdgQiqcpQ5BG/Q== {MON’s IP}:/ /mnt/mycephfs
```
