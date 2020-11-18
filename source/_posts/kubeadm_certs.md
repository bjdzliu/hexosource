---
title: Kubeadm_证书
date: 2019-11-12 07:11:41
tags: k8s
categories:
- Technical Notes
---
### Kubeadm_证书
介绍kubeadm中证书的用法，权限以及自定义证书的注意事项。

kubeadm启动后生成如下证书:

>ls -l /etc/kubernetes/pki  
-rw——- 1 root root 1679 Sep 6 14:12 ca.key  
-rw-r–r– 1 root root 1025 Sep 6 14:12 ca.crt    
-rw——- 1 root root 1675 Sep 6 14:12 apiserver.key  
-rw-r–r– 1 root root 1229 Sep 6 14:12 apiserver.crt  
-rw——- 1 root root 1675 Sep 6 14:12 apiserver-kubelet-client.key  
-rw-r–r– 1 root root 1099 Sep 6 14:12 apiserver-kubelet-client.crt  
-rw——- 1 root root 451 Sep 6 14:12 sa.pub  
-rw——- 1 root root 1675 Sep 6 14:12 sa.key  
-rw——- 1 root root 1679 Sep 6 14:12 front-proxy-ca.key  
-rw-r–r– 1 root root 1025 Sep 6 14:12 front-proxy-ca.crt  
-rw——- 1 root root 1679 Sep 6 14:12 front-proxy-client.key  
-rw-r–r– 1 root root 1050 Sep 6 14:12 front-proxy-client.crt  

每个组件(kubelet ,proxy ,controller-manager, scheduler)使用不同的conf文件

>/etc/kubernetes/admin.conf
/etc/kubernetes/kubelet.conf
/etc/kubernetes/controller-manager.conf
/etc/kubernetes/scheduler.conf

后面会对kubelet.conf和controller-manager.conf进行分析

以上文件中的data属性，都是base64编码,解码后,放在一个文件中，用openssl查看

```
openssl req -text -noout -in server.csr
openssl x509 -noout -text -in /srv/kubernetes/ca.crt
```

kubeadm init后提示进行环境变量的设置，就是让kubectl 拥有admin的权限：  
export KUBECONFIG=/tmp/admin.conf  
kubectl使用admin.conf进行连接,也是一个客户端  

每个conf文件中都有根证书，内容都是一样：  
>apiVersion: v1  
clusters:  
– cluster:  
certificate-authority-data:  
........


决定组件连接到哪个apiserver有两个参数：  
组件：kube-scheduler  
>–kubeconfig string Path to kubeconfig file with authorization and master location information.    
–master string The address of the Kubernetes API server (overrides any value in kubeconfig)  

组件：kubelet
>–kubeconfig string Path to a kubeconfig file, specifying how to connect to the API server. (default “/var/lib/kubelet/kubeconfig”)

组件:kube-proxy
>–kubeconfig string Path to kubeconfig file with authorization information (the master location is set by the master flag).  
–master string The address of the Kubernetes API server (overrides any value in kubeconfig)

组件:kube-controller,有三个参数，注意下:
>–kubeconfig string Path to kubeconfig file with authorization and master location information.  
–cluster-signing-cert-file string Filename containing a PEM-encoded X509 CA certificate used to issue cluster-scoped certificates (default “/etc/kubernetes/ca/ca.pem”)  
–cluster-signing-key-file string Filename containing a PEM-encoded RSA or ECDSA private key used to sign cluster-scoped certificates (default “/etc/kubernetes/ca/ca.key”)  

启动API Server RBAC时, 先通过kubeconfig认证 Authentication ,如果指定了以上的后两个参数,就通过以上两个参数,进行授权 Authrizatoin 验证  

以下是 kubelet controller 和admin 使用的kubeconfig参数里的配置文件参数
除了api.crt 是 TLS Web Server Authentication , 其他是 TLS Web Client Authentication  

*kubelet.conf ,controller-manager.conf 和 admin.conf* 都被认为是API Server的客户端，都包含字段
```
cat kubelet.conf:
– name: system:node:ubuntu-102A
user:
client-certificate-data:
把字段的内容经过base64解码，保存在kubelet.crt中
```

```
cat controller-manager.conf
– name: system:kube-controller-manager
user:
client-certificate-data:
把字段 的内容经过base64解码，保存在controllet.crt中
```

```
cat admin.conf
– name: kubernetes-admin
user:
client-certificate-data:
把字段 的内容经过base64解码，保存在admin.crt 中
```

看看以上的crt文件中的内容是什么:
>CN : 作为用户名  
kubelet.crt 摘自kubelet.con  
Subject: O=system:nodes, CN=system:node:ubuntu-102A  
X509v3 extensions:  
X509v3 Key Usage: critical  
Digital Signature, Key Encipherment  
X509v3 Extended Key Usage:  
TLS Web Client Authentication  

controllet.crt 摘自controller-manager.conf
>Subject: CN=system:kube-controller-manager  
X509v3 extensions:
X509v3 Key Usage: critical  
Digital Signature, Key Encipherment  
X509v3 Extended Key Usage:    
TLS Web Client Authentication  

admin.crt 摘自admin.conf
>Subject: O=system:masters, CN=kubernetes-admin  
X509v3 extensions:  
X509v3 Key Usage: critical    
Digital Signature, Key Encipherment  
X509v3 Extended Key Usage:  
TLS Web Client Authentication  
/*** 如果CN=system:masters:kubernetes-admin 显示用户system:masters:kubernetes-admin没权限. *** /

client端证书中的CN name 决定了用户的角色。[参考官方文档](https://kubernetes.io/docs/admin/authorization/rbac/ )

>Core Component Roles：  
system:kube-scheduler  
system:kube-controller-manager  
system:node  
system:node-proxie  

查看 /etc/kubernetes/pki 路径下的apiserver.crt 证书内容  
查看apiserver的证书：
```
openssl x509 -noout -text -in apiserver.crt
```

>X509v3 extensions:  
X509v3 Key Usage: critical  
Digital Signature, Key Encipherment  
X509v3 Extended Key Usage:  
TLS Web Server Authentication  
X509v3 Subject Alternative Name:  
DNS:ubuntu-102A, DNS:kubernetes, DNS:kubernetes.default,   DNS:kubernetes.default.svc, DNS:kubernetes.default.svc.cluster.local, IP Address:10.96.0.1, IP Address:9.125.72.207  

查看 ca.crt 的证书，他的CN名字叫kubernetes  
**Subject: CN=kubernetes**

默认情况下，k8s启动匿名访问(从1.6开始)，
```
root@ubuntu430:/tmp# curl https:/masternode:6443 -k
User “system:anonymous” cannot get at the cluster scope.root@ubuntu430:/tmp#
```

因为client证书中使用了：  
**X509v3 Extended Key Usage:  
TLS Web Client Authentication**  
所以，就不用在client端的证书中添加X509v3 Subject Alternative Name，
目的是：  
client的ip，hostname，domain name可以是任意。  
同理，server证书也使用了：  
**X509v3 Extended Key Usage:  
TLS Web Server Authentication**

通过手工搭建3台HA的k8s集群，做证书时使用了文件：
```
root@master01:/srv/kubernetes# cat extfile.cnf
subjectAltName = IP.1:10.0.0.1,IP.2:192.168.100.170,IP.3:127.0.0.1 ,IP.5:10.96.0.1,DNS.3:kubernetes.default,DNS.4:*.cibfintech.com,DNS.5:kubernetes.default.svc.cibfintech.com,DNS.6:kubernetes.default.svc,DNS.7:kubernetes,DNS.8:*.baasclusteronz.local
```

- openssl的关键内容:

```
[ CA_default ]

dir = ./ # Where everything is kept  
certs = $dir/certs # Where the issued certs are kept  
crl_dir = $dir/crl # Where the issued crl are kept  
database = $dir/index.txt # database index file.  
# unique_subject = no # Set to ‘no’ to allow creation of  
# several ctificates with same subject.
new_certs_dir = $dir/newcerts # default place for new certs.

certificate = $dir/ca.crt # The CA certificate
serial = $dir/serial # The current serial number
crlnumber = $dir/crlnumber # the current crl number
# must be commented out to leave a V1 CRL
crl = $dir/crl.pem # The current CRL
private_key = $dir/ca.key # The private key
RANDFILE = $dir/private/.rand # private random number file

x509_extensions = usr_cert # The extentions to add to the cert
```
***

**问题:**
第二次生成client.crt时，报错。  
因为序号的问题  
Sign the certificate? [y/n]:y  
failed to update database  
TXT_DB error number 2  
解决:  
echo “unique_subject = no ” > index.txt.attr  
or  
rm index.txt  

**csr的请求信息要和ca的一致**

***
实验，在一台客户端的机器上制作证书，API Server端没启用RBAC  
```
openssl req -new -key client.key -out client2.csr -config openssl-client.cnf
openssl ca -keyfile ./ca.key -cert ./ca.crt -config ./openssl-client.cnf -out client2.crt -infiles client2.csr #以config为准
```
如果在openssl-client.cnf中，设置了ca的证书和key:
```
openssl ca -config ./openssl.cnf -out client2.crt -infiles client2.csr
```


`curl https://172.16.32.131 –cert /srv/kubernetes/client2.crt –key /srv/kubernetes/client.key -k`

查看client2.crt证书的内容，没启用RBAC，CN的名字随意  
```
root@vm-1494481207803:/srv/kubernetes# openssl x509 -text -noout -in client2.crt
Certificate:
Data:
Version: 3 (0x2)
Serial Number: 1 (0x1)
Signature Algorithm: sha256WithRSAEncryption
Issuer: C=CN, ST=BJ, L=BJ, O=xingye, OU=baas, CN=k8scaserver/emailAddress=userbaas@cn.ibm.com
Validity
Not Before: Jul 12 15:45:43 2017 GMT
Not After : Jul 12 15:45:43 2018 GMT
Subject: C=CN, ST=BJ, O=xingye, OU=Default Company Ltd, CN=k8s142
Subject Public Key Info:
Public Key Algorithm: rsaEncryption
Public-Key: (2048 bit)
Modulus:
00:c9:c5:bc:56:b2:2a:70:27:2c:e0:0c:87:e2:15:
02:e6:b5:a9:cf:f8:40:45:82:84:55:fb:e3:35:3e:
Exponent: 65537 (0x10001)
X509v3 extensions:
X509v3 Basic Constraints:
CA:FALSE
Netscape Cert Type:
SSL Client, SSL Server
Netscape Comment:
OpenSSL Generated Certificate
X509v3 Subject Key Identifier:
A3:F8:0B:49:11:36:58:16:B3:58:55:76:35:81:22:3A:68:16:9A:D6
X509v3 Authority Key Identifier:
keyid:03:B9:05:9A:63:F1:56:2F:6A:66:1D:B3:5B:2F:F4:FF:D3:F9:98:69

X509v3 Extended Key Usage:
TLS Web Server Authentication, TLS Web Client Authentication, Code Signing, E-mail Protection
X509v3 Key Usage:
Digital Signature, Non Repudiation, Key Encipherment
Signature Algorithm: sha256WithRSAEncryption
b7:89:e6:3e:46:c7:1f:a3:47:43:32:e3:d7:f2:08:34:44:b9:
```

结论: 不使用altname,也可以进行认证
blockchain所有的证书都是用这种方式，CN名字: peer0.xxxxx

参考学习，方法都很简便：  
https://access.redhat.com/solutions/28965  
https://kubernetes.io/docs/concepts/cluster-administration/certificates/  

***
实验， kubeadm拉起的cluster,如果想修改cluster的IP

自己做一个api.crt，确定alter name：  
>DNS:ubuntu430, DNS:kubernetes, DNS:kubernetes.default, DNS:kubernetes.default.svc, DNS:kubernetes.default.svc.cluster.local, IP Address:10.96.0.1, IP Address:192.168.0.22, IP Address:127.0.0.1

生成key和cert：
```
openssl req -new -key apiserver.key -out apiserver.csr -config openssl.cnf
root@ubuntu430:/etc/kubernetes/pki-local/1# openssl ca -keyfile ./ca.key -cert ./ca.crt -config ./openssl.cnf -out apiserver.crt -infiles apiserver.csr
Using configuration from ./openssl.cnf
Check that the request matches the signature
Signature ok
```

用证书apiserver.crt, 替换原来的/etc/kubernetes/pki/apiserver.crt

修改 */etc/kubernetes/* 下所有和 *192.168.0.44* 相关的IP 为 *192.168.0.22*

修改proxy的启动配置文件
kubectl edit cm kube-proxy -o yaml -n kube-system

总结：api的证书替换后，其他组件和组件使用的conf中使用新的IP。
