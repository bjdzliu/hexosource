---
title: helm快速理解
date: 2017-12-5 21:08:57
tags: k8s
categories:
- Technical Notes
---
### Helm快速理解
官方文档：[地址](https://helm.sh/docs/intro/)

概念：
- 类似apt yum
- 有一个stable仓库（网络上的），一个本地仓库（自己创建）
- helm-server使用k8s的api部署应用
- helm客户端操作charts

1. 创建了一个本地仓库，并且在本地的kubectl config view里，创建了helm-server–> tiller  
```
root@zbrand-T430:~/liudz/helm/linux-amd64# ./helm init  
Creating /root/.helm  
Creating /root/.helm/repository  
Creating /root/.helm/repository/cache  
Creating /root/.helm/repository/local
Creating /root/.helm/plugins  
Creating /root/.helm/starters
Creating /root/.helm/cache/archive
Creating /root/.helm/repository/repositories.yaml
Adding stable repo with URL: https://kubernetes-charts.storage.googleapis.com
```

2. 为tiller授权 in K8S
```
kubectl create serviceaccount –namespace kube-system tiller
kubectl create clusterrolebinding tiller-cluster-rule –clusterrole=cluster-admin –serviceaccount=kube-system:tiller
kubectl patch deploy –namespace kube-system tiller-deploy -p ‘{“spec”:{“template”:{“spec”:{“serviceAccount”:”tiller”}}}}’
```


3. 启动本地的仓库监听端口
```
helm server –address 0.0.0.0:8879
```

4. 基本命令  
– 查看可用的charts包  
helm search  
– 更新charts列表  
helo repo update  
– 查看集群内安装：  
helm list  

5. 部署发布
```
helm search jenkins
helm install –name my-release –set Persistence.StorageClass=slow stable/jenkins
```

6. 创建一个自己的chart
```
helm create mychart //当前目录下生成mycharts
```

7. 上传到本地仓库
- 启动监听，目的是为了其他helm客户端，也可以使用
helm serve –address 0.0.0.0:8879

- 另一个窗口中，打包
```
root@zbrand-T430:~# helm package ./mychart
Successfully packaged chart and saved it to: /root/mychart-0.1.0.tgz
```
- 查看本地仓库，默认读取的是/root/.helm里的配置
```
root@zbrand-T430:~# helm search local
NAME VERSION DESCRIPTION
local/mychart 0.1.0 A Helm chart for Kubernetes
```
8. 安装  
如果未启动 server： 8879
```
helm install –name example5 ./mychart –set service.type=NodePort
or
helm install –name example3 mychart-0.1.0.tgz –set service.type=NodePort
```
如果启动了 server： 8879  
```
helm install –name example4 local/mychart –set service.type=NodePort
```
测试：
```
export NODE_PORT=$(kubectl get –namespace default -o jsonpath=”{.spec.ports[0].nodePort}” services example4-mychart)
export NODE_IP=$(kubectl get nodes –namespace default -o jsonpath=”{.items[0].status.addresses[0].address}”)
echo http://$NODE_IP:$NODE_PORT/
```
浏览器里，访问：http://9.115.114.218:31728/

9. 分析mychart
预部署测试的输出：

  ```
  helm install –dry-run –debug ./mychart –set service.internalPort=8080  
  [debug] Created tunnel using local port: ‘37381’  

  [debug] SERVER: “127.0.0.1:37381”  

  [debug] Original chart version: “”  
  [debug] CHART PATH: /root/mychart  

  NAME: limping-hare  
  REVISION: 1  
  RELEASED: Tue Dec 5 15:20:58 2017
  CHART: mychart-0.1.0
  USER-SUPPLIED VALUES:
  service:
  internalPort: 8080
  ```

  COMPUTED VALUES:    

  ```
  # Source: mychart/templates/service.yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: limping-hare-mychart
    labels:
      app: mychart
      chart: mychart-0.1.0
      release: limping-hare
      heritage: Tiller
  spec:
    type: ClusterIP
    ports:
      - port: 80
        targetPort: 8080
        protocol: TCP
        name: nginx
    selector:
      app: mychart
      release: limping-hare
  ```

  通过输出对比原始的模板文件，这里以其中的services为例：  
  File： ~/mychart/Chart.yaml

  ```
  apiVersion: v1
  description: A Helm chart for Kubernetes
  name: mychart
  version: 0.1.0
  ```

  File： ~/mychart/values.yaml 包含默认值，会传递到template中，截取其中一段说明：

  ```
  service:
    name: nginx
    type: ClusterIP
    externalPort: 80
    internalPort: 80
  ```

  File：~/mychart/templates/service.yaml

  ```
  apiVersion: v1
  kind: Service
  metadata:
    name: {{ template "mychart.fullname" . }}    ---- > name: limping-hare-mychart
    labels:
      app: {{ template "mychart.name" . }}       ---------> app: mychart  template 是内置函数
      chart: {{ .Chart.Name }}-{{ .Chart.Version | replace "+" "_" }}  ----> chart: mychart-0.1.0
      release: {{ .Release.Name }}    ---------> release: limping-hare   Release是内置对象
      heritage: {{ .Release.Service }}
  spec:
    type: {{ .Values.service.type }}
    ports:
      - port: {{ .Values.service.externalPort }}
        targetPort: {{ .Values.service.internalPort }}
        protocol: TCP
        name: {{ .Values.service.name }}
    selector:
      app: {{ template "mychart.name" . }}
      release: {{ .Release.Name }}
  ```
