---
title: INGRESS EXPERIMENT
date: 2019-10-29 05:44:59
tags: k8s
categories:
- Technical Notes
---
本文介绍了ingress 实践。

官方文档：[地址](https://kubernetes.io/docs/concepts/services-networking/ingress/)
1. 创建service  
创建3个service，实际应用中可能不止这三个。
创建ingress

```

kubectl create ns ingress-nginx
```

创建后端服务1，并暴露出端口

```
kubectl run echoheaders --image=gcr.io/google_containers/echoserver:1.4 --replicas=1 --port=8080 -n ingress-nginx
kubectl expose deployment echoheaders --port=80 --target-port=8080 --name=echoheaders-x -n ingress-nginx
//提供default service
```

创建后端服务2：tomcat service  

```
kubectl run tomcat --image=tomcat:docs --replicas=1 --port=8080 -n ingress-nginx
kubectl expose deployment tomcat --port=80 --target-port=8080 --name=tomcat-x -n ingress-nginx  

//自制镜像tomcat:docs，将webapps.list中的应用（ROOT  docs  examples  host-manager  manager）都copy到webapps中
```

创建后端服务3：nginx service

```
kubectl run nginx --image=nginx --replicas=1 --port=80 -n ingress-nginx
kubectl expose deployment nginx  --port=80 --target-port=80 --name=nginx-x -n ingress-nginx
//代表其他服务
```
2. 创建ingress rule  
ingress rule 可以有多个，创建后，不需要重启nginx-controller。  
  - controller怎么识别到ingress rule?    
     controller的启动参数默认指定class 为ingress。同时如果没有class的ingress rule，也会由controler负责使其生效。通过这种方式，可以实现多个controller对应实现不同的insgress rule。  
我们依次创建2个rule:  
RULE-1:  

  ```
  apiVersion: extensions/v1beta1
  kind: Ingress
  metadata:
    name: tomcat-root-ingress
    annotations:
      ingress.kubernetes.io/rewrite-target: /
  spec:
    rules:
    - http:
        paths:
        - path: /
          backend:
            serviceName: tomcat-x
            servicePort: 80
  ```

  域名：http://192.168.37.112/  
  根路径访问，转发到 tomcat-x 服务的ROOT应用。  
  针对 / , nginx-controller自动配置了https转发：  

  ```
  if ($redirect_to_https) {

        return 308 https://$best_http_host$request_uri;

      }
  ```

RULE-2:  

  ```
  apiVersion: extensions/v1beta1
  kind: Ingress
  metadata:
    name: tomcat-ingress
    namespace: ingress-nginx
    annotations:
      ingress.kubernetes.io/rewrite-target: /
  spec:
    rules:
    - http:
        paths:
        - path: /docs
          backend:
            serviceName: tomcat-x
            servicePort: 80
      host: tomcat.bar.com
  ```
  域名：tomcat.bar.com  
  路径： /docs  
  转发到tomcat-x的docs应用里。

3. 创建 nginx-controller  

  ```
  wget https://github.com/kubernetes/ingress-nginx/blob/nginx-0.20.0/deploy/with-rbac.yaml
  kubectl apply -f with-rbac.yaml

  ```

  注：执行前，手工修改with-rbac.yaml,
      -  添加hostNetwork: true。
        为了本次测试，nginx-controller在一台node节点上启动，并对外提供服务。生产上一般准备3个node对外服务，或者前面放一个svc绑定slb。
      -  设置默认的default service，专门处理ingress rule没有匹配的情况。

          ```
           - --default-backend-service=$(POD_NAMESPACE)/echoheaders-x
          ```
4. 效果  
  本地电脑绑定tomcat.bar.com解析到vm ip 192.168.37.112。  

  - 访问 http://tomcat.bar.com/docs --- ok 仍旧是http
  - 访问 http://tomcat.bar.com/fakeserver --- 返回 default backend - 404

  - 访问 http://192.168.37.112/ ---redirect_to_http--> https://192.168.37.112/
    - 证书是nginx-controller中配置的  *default-fake-certificate.pem*

  - 访问 https://192.168.37.112/asdasd -- Tomcat返回404
