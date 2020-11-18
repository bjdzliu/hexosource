---
title: k8s 自动扩缩容概览
date: 2020-11-18 16:16:51
tags:
- k8s
categories:
- Technical Notes
---
### k8s 自动扩缩容概览

##### Overview
![Overview](
https://bjdzliu.oss-cn-beijing.aliyuncs.com/hexo_images/k8s_autoscaler/image1.png)

Cluster Autoscaler - a component that automatically adjusts the size of a Kubernetes Cluster so that all pods have a place to run and there are no unneeded nodes. Works with GCP, AWS and Azure. Version 1.0 (GA) was released with kubernetes 1.8.
当前项目中使用该组件，组件和k8s有版本匹配的关系。

![版本对应关系](
https://bjdzliu.oss-cn-beijing.aliyuncs.com/hexo_images/k8s_autoscaler/image2.png)

[详细介绍](https://github.com/kubernetes/autoscaler/tree/cluster-autoscaler-release-1.18/cluster-autoscaler)


**_Vertical Pod Autoscaler_** - a set of components that automatically adjust the amount of CPU and memory requested by pods running in the Kubernetes Cluster. Current state - beta.


**_Addon Resizer_** - a simplified version of vertical pod autoscaler that modifies resource requests of a deployment based on the number of nodes in the Kubernetes Cluster. Current state - beta.
[Addon Resize详细介绍](https://github.com/kubernetes/autoscaler/tree/addon-resizer-release-1.8/addon-resizer)：

*Charts - Supported Helm charts for components above.*

**_HPA_**  
Horizontal Pod Autoscaler automatically scales the number of Pods in a replication controller, deployment, replica set or stateful set based on observed CPU utilization (or, with beta support, on some other, application-provided metrics).
[HPA详细介绍](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)


##### 术语
up 和 down 意味着保持相同数量的实例，但增加或减少 CPU 或 Server 的内存。  
In 和 out 是增加或删除服务实例，但保持资源不变。


##### 监控/扩缩容各组件关系
![components](
https://bjdzliu.oss-cn-beijing.aliyuncs.com/hexo_images/k8s_autoscaler/image3.png)


比如在aks中的组件：
![azure](
https://bjdzliu.oss-cn-beijing.aliyuncs.com/hexo_images/k8s_autoscaler/cluster-autoscaler4.png)

[azure doc](https://docs.azure.cn/zh-cn/aks/concepts-scale#cluster-autoscaler)

K8s的API： TBD

##### Cluster Autoscaler （CA）
In another artical

##### HPA
前提：启用了metric-server
Metrics Server .note

通过命令方式启动：
kubectl autoscale deployment azure-vote-front --cpu-percent=50 --min=3 --max=10
或者，通过yaml文件自定义：
![yaml file](
https://bjdzliu.oss-cn-beijing.aliyuncs.com/hexo_images/k8s_autoscaler/image5.png)

**_HPA的算法:_**  
期望副本数 = 向上取整 ( 现在测量值 / 期望测量值 )
现在测量值往往是所有副本测量值的平均值
期望测量值则往往是所有副本Requests值 * 缩扩容目标值.
（缩扩容目标值在创建HPA资源时定义）

期望副本数 = 向上取整 ( sum ( 测量值 ) / sum ( Requests ) * 缩扩容目标值)
当前的副本数是2, 指定的Requests为cpu: 100m, 若现在的测量值为cpu: 200m, 缩扩容目标值为50%, 则:
期望副本数 = ceil(200/(100*50%)) = ceil(4) = 4

targetCPUUtilizationPercentage:
值越低，说明期望得到的pod 越多。

查看hpa配置：
kubectl get hpa

HPA支持的版本
*autoscaling/v1、autoscaling/v2beta1和autoscaling/v2beta2*  三个大版本。  
autoscaling/v1 ---  CPU only
autoscaling/v2beta1 --- Support Custom Metric  
autoscaling/v2beta2 ----- 额外增加了外部指标支持  

控制HPA
controller mamaner参数：  
*-horizontal-pod-autoscaler-sync-period* 确定 HPA 对于Pods组指标的监控频率。默认的周期为30秒。  
*-horizontal-pod-autoscaler-upscale-delay* 两次扩展操作之间的默认间隔为3分钟  
*-horizontal-pod-autoscaler-downscale-delay* 两个缩小操作之间的默认间隔为5分钟  

官方链接：  
https://github.com/kubernetes/autoscaler
https://github.com/kubernetes/community/blob/master/contributors/design-proposals/instrumentation/custom-metrics-api.md
