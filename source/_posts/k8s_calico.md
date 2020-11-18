---
title: K8S-CALICO-网络验证
date: 2017-11-08 06:16:31
tags:
- k8s
categories:
- Technical Notes
---
### K8s-Calico-网络验证
k8s和calico已经深度结合，在使用上也很方便。实验内容来自双方的官方文档。
1. 默认可以互相访问  
比如：
```
kubectl run nginx --image=nginx --replicas=2
kubectl expose deployment nginx --port=80
kubectl run busybox --rm -ti --image=busybox /bin/sh
/ # wget --spider --timeout=1 nginx
Connecting to nginx (10.108.104.102:80)
```

  其他namespace也可以访问  
```
kubectl run busybox –rm -ti –image=busybox /bin/sh -n liudz1  
/ # wget –spider –timeout=1 nginx.default  
Connecting to nginx (10.108.104.102:80)  
```

2.  限制访问，只有指定标签才可以访问
```
cat <<EOF > NetworkPolicy.yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: access-nginx
spec:
  podSelector:
    matchLabels:
      run: nginx
  ingress:
  - from:
    - podSelector:
        matchLabels:
          access: "true"
EOF
```

  创建策略
```
kubectl create -f NetworkPolicy.yaml
```

  测试  
```
kubectl run busybox --rm -ti --image=busybox /bin/sh
/ # wget --spider --timeout=1 nginx
Connecting to nginx (10.108.104.102:80
wget: download timed out
```

  加上指定标签就可以访问：  
```
kubectl run busybox --rm -ti  --labels="access=true" --image=busybox /bin/sh
/ # wget --spider --timeout=1 nginx
Connecting to nginx (10.108.104.102:80)
```
3. 给表空间创建默认的禁止规则，隔离baas和用户的网络  
  - 3.1 首先为每一个表空间创建隔离策略

  ```
  cat <<EOF > default-deny.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
  - Ingress
EOF
  ```
  创建全部拒绝策略

  ```
  kubectl create -f default-deny.yaml -n liudz1
  ```
  - 3.2 baas表空间创建的pod全部带上run:baas

  ```
  kubectl run nginx --image=nginx --labels="run=baas"  --replicas=1 -n liudz1
  ```
    即使在同一个表空间下启动一个busybox，也不可以访问nginx
    ```
    kubectl run busybox --rm -ti --image=busybox /bin/sh -n liudz1
    kubectl expose deployment nginx --port=80  -n liudz1
    / #  wget --spider --timeout=1 nginx
    Connecting to nginx (10.99.110.2:80)
    wget: download timed out
    ```

  - 3.3  允许表空间内的的pod访问指定label的pod,其他表空间不能访问
  ```
  kubectl create -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: access-nginx2
  namespace: liudz1
spec:
  podSelector:
    matchLabels:
      run: baas
  ingress:
    - from:
      - podSelector:
          matchLabels: {}
EOF
  ```

    同一个表空间可以访问：
  ```
  kubectl run busybox --rm -ti --image=busybox   /bin/sh -n liudz1
/ #  wget --spider --timeout=1 nginx
Connecting to nginx (10.99.110.2:80)
  ```

    不同表空间不可以访问:
```
kubectl run busybox --rm -ti --image=busybox   /bin/sh
/ # wget --spider --timeout=1 nginx.liudz1
Connecting to nginx.liudz1 (10.99.110.2:80)
wget: download timed out
```
  - 3.4 创建一个允许全部访问的规则，这个规则覆盖掉之前的隔离规则，其他表空间也可以访问liudz1
    ```
      kubectl create -f - << EOF
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: allow-all
      namespace: liudz1
    spec:
      podSelector: {}
      ingress:
      - {}
    EOF
    networkpolicy "allow-all" created
    ```

  - 3.5 允许表空间内的的pod互相访问，但其他表空间的pod不能来访问

    ```
    kubectl create -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: access-nginx
  namespace: liudz1
spec:
  podSelector:
    matchLabels: {}
  ingress:
    - from:
      - podSelector:
          matchLabels: {}
EOF
```

    ```
    root@ubuntubase:~# kubectl run busybox2  --rm -ti --image=busybox   /bin/sh -n liudz1
    If you don't see a command prompt, try pressing enter.
    / #  wget --spider --timeout=1 nginx.liudz1
    Connecting to nginx.liudz1 (10.99.110.2:80)
    ```

    ```
    root@ubuntubase:~# kubectl run busybox2  --rm -ti --image=busybox   /bin/sh  
    If you don't see a command prompt, try pressing enter.  
    / #  wget --spider --timeout=1 nginx.liudz1  
    wget: download timed out
    ```

    通过指定labels，并满足在同一个namespace，才能访问
    ```
    kubectl create -f - <<EOF
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: access-nginx
      namespace: liudz1
    spec:
      podSelector:
        matchLabels: {}
      ingress:
        - from:
          - podSelector:
              matchLabels:
                access: ceshi1
    EOF
    ```

    不在同一个namespace，无法访问
```
kubectl run busybox2  --rm -ti --labels="access=ceshi1" --image=busybox   /bin/sh
 / # wget --spider --timeout=1 nginx.liudz1
Connecting to nginx.liudz1 (10.99.110.2:80)
wget: download timed out
```

    在一个namespace，没有对应的label，无法访问
```
root@ubuntubase:~# kubectl run busybox2  --rm -ti --image=busybox   /bin/sh -n liudz1
If you don't see a command prompt, try pressing enter.
/ # wget --spider --timeout=1 nginx.liudz1
Connecting to nginx.liudz1 (10.99.110.2:80)
wget: download timed out
```

    在一个namespace，有对应的label，可以访问
```
kubectl run busybox2  --rm -ti --labels="access=ceshi1" --image=busybox   /bin/sh  -n liudz1
/ # wget --spider --timeout=1 nginx.liudz1
Connecting to nginx.liudz1 (10.99.110.2:80)
```
  - 3.6 试验 允许liudz2 表空间可以访问任何其他表空间
namespace1 namespace2两者的网络策略目前如下：
```
root@ubuntubase:~# kubectl get netpol -n liudz1
NAME           POD-SELECTOR   AGE
default-deny   <none>         6h
root@ubuntubase:~# kubectl get netpol -n liudz2
No resources found.
```

    给liudz2打label
    ```
    kubectl label namespace liudz2 name=liudz2

    kubectl create -f - <<EOF
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: access-nginx
      namespace: liudz1
    spec:
      podSelector: {}
      ingress:
        - from:
          - namespaceSelector:
              matchLabels:
                name: liudz2
    EOF
    ```

    其他表空间不能访问：
```
root@ubuntubase:~# kubectl run busybox2  --rm -ti --image=busybox   /bin/sh
If you don't see a command prompt, try pressing enter.
/ # wget --spider --timeout=1 nginx.liudz1
Connecting to nginx.liudz1 (10.99.110.2:80)
wget: download timed out
```

    具有label的namespace liudz2可以访问：
```
root@ubuntubase:~# kubectl run busybox2  --rm -ti --image=busybox   /bin/sh -n liudz2
If you don't see a command prompt, try pressing enter.
/ # wget --spider --timeout=1 nginx.liudz1
Connecting to nginx.liudz1 (10.99.110.2:80)
```

    自己namespace也不能访问：
```
root@ubuntubase:~# kubectl run busybox2  --rm -ti --image=busybox   /bin/sh -n liudz1
If you don't see a command prompt, try pressing enter.
/ # wget --spider --timeout=1 nginx.liudz1
Connecting to nginx.liudz1 (10.99.110.2:80)
wget: download timed out
```

    只允许来自label为name: liudz2 和 name: liudz1的访问
```
kubectl create -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: access-nginx
  namespace: liudz1
spec:
  podSelector: {}
  ingress:
    - from:
      - namespaceSelector:
          matchLabels:
            name: liudz2
      - namespaceSelector:
          matchLabels:
            name: liudz1
EOF
```

    其他namespace不能访问
```
root@ubuntubase:~# kubectl run busybox2  --rm -ti --image=busybox   /bin/sh
If you don't see a command prompt, try pressing enter.
/ # wget --spider --timeout=1 nginx.liudz1
Connecting to nginx.liudz1 (10.99.110.2:80)
exwget: download timed out
```
