---
title: aliyun_summary1
date: 2019-07-21 13:21:30
tags: aliyun
categories:
- Technical Notes
---
包含十个问题。
以简短的方式，介绍使用阿里云过程中遇到的问题。
部分回答，来自阿里支持2线、3线。


#### 问题1
##### 描述：  
有一台ecs连接slb，超时。  
ecs上没有工作负载。  
slb ip：87.65.43.21  
slb id: lb-2jd9xxxxsg2xxxxx2  
ecs在vpc网络里，访问 onetest.citic.com 对应的slb ip：现象如下：   
![snap1](https://bjdzliu.oss-cn-beijing.aliyuncs.com/hexo_images/aliyun_QA/aliyun-1)
访问baidu，很快就返回了。
![snap2](https://bjdzliu.oss-cn-beijing.aliyuncs.com/hexo_images/aliyun_QA/aliyun-2)

##### 过程：
检查出提供外网访问的nat服务器，cms监控显示出口带宽已经达到峰值。
改nat服务器，用于全网的出外网IP。

##### 结论：
扩容出口nat服务器(使用iptables)的带宽为100Mb

#### 问题2
##### 描述：
17个节点组成swarm集群，前端SLB-前端连接数20万，监听http 80，配置了7层路由转发。压力测试时，并发5万，报大量的503错误。
![snap3](https://bjdzliu.oss-cn-beijing.aliyuncs.com/hexo_images/aliyun_QA/aliyun-3)

##### 结论
我们这边看按到SLB上有限速丢弃的记录，应该是压测的时候有瞬间的连接突增导致瞬间的并发连接已经超过20万，或者新建超过2万。  
说一下我们数据采集的原理：  
slb的的并发连接数量是每分钟采样一次，抓去当时连接的数量。一瞬间的并发暴增可能在采样间隔期间，导致看不到。  
而新建连接和限速丢弃连接是一个累计值，每当有新建/丢弃就会+1，然后每分钟取一次和上一分钟的差值。因此只要有新建/丢弃就会记录下来，不会漏记。  
比如我们在做压测的时候，压测客户端开启20万并发连接，客户端在一瞬间会建立起大量的连接，可能导致新建连接超时。  
以上是15点到16点之间TCP80端口的情况  

再看配置了http转发规则的81端口，压测时间是16点50和17点37两波，这个slb的规格是http qps是20000，但是压测峰值已经到18万qps，所以大量的限速丢弃导致503  

通过查看CMS监控：
![snap4](https://bjdzliu.oss-cn-beijing.aliyuncs.com/hexo_images/aliyun_QA/aliyun-4)

#### 问题3
##### 问:
阿里云中redis集群版用lua脚本有没有什么限制？
##### 答:
[link](https://help.aliyun.com/document_detail/26356.html)

#### 问题4
##### 问:
相比较redis、guava等锁，用lua控制库存的性能怎样？
##### 答:
对比使用redis做锁和使用lua直接控制库存来说，使用redis做分布式锁的逻辑多次的网络交互（1. 与redis交互锁，2. 实现库存增删，3.释放锁等），而使用lua等于是在redis本地运行一个lua脚本，直接操作内存，等于是减少了很多的网络交互和本地的堆栈调用，所以lua的性能肯定是更好的.  
guava的锁我没太了解，但是首先原理上是类似的，其次也要看redis的性能和guava的性能，应该也会有一些差距

#### 问题5
##### 问:
防羊毛党的有哪些手段？

##### 答:
防薅羊毛的方法有很多，根据业务不同，场景不同，风险不同可以选择多种方式进行防护，从部署难易程度和成本综合评价主要包括以下几种： 1、增加验证码方式，使用WAF风控模块，进行基本的人机识别，过滤掉大部分爬虫类或机器人的访问行为--针对一般水平初级的黑产； 2、使用手机、IP等信誉库，例如阿里云可以输出的，经过长期积累并可实时更新的黑产手机号，在交易环节验证用户风险---针对有一定组织能力手中有大量手机号或肉鸡资源的黑产； 3、根据具体业务情况，制定有针对性的虚假交易发现机制，需要业务流程配合进行一定的改造---针对比较专业，面向特殊行业有高额获利能力的专业黑产或行业内的恶意竞争。


#### 问题6
##### 问:
秒杀活动除了接口压测外，还有哪些手段测试系统性能？
##### 答：
只有通过压测了解性能，关键是要制定流量控制策略，避免雪崩，从前到后，流量逐步减少

#### 问题7
##### 问:
秒杀活动开始按钮点亮有哪些方式？
##### 答：
点亮按钮没什么好办法。大秒是提前几秒或几分钟点亮；要做到精确有点难度。或者适当提前一点点，点了服务器没开始，也不影响用户体验。

#### 问题8
##### 问:
阿里云只读rds同步读写实例出现问题，阿里云是怎么恢复同步的

##### 答：
一般真的同步出现问题 我们采取的策略都是重搭
重搭数据从哪里来？
主库 全量+增量 【主库的备份集+binlog】
问：用户如果没有开启日志备份呢
不可能。备份策略里这是强制的
如果用户的备份策略是一周备份一次，那就有可能出现全量用一周前的备份集恢复
再拉取近一周binlog增量 这是最极端情况 这种情况下 恢复就会比较慢
日志备份开启指的上传OSS
假设有个极端场景 备份集是上周的 但是binlog中间的被删除了
那我们RDS会在恢复前 重新做一次主库的备份集
再用新做的备份集 + Binlog恢复
所以您不用担心这里的冗余机制。。。
重搭库，用户无知晓。

#### 问题9
##### 问：
swarm容器集群，某个应用有10个容器，执行重新调度的过程中，对外服务有中断，表现为服务链接超时。重新调度或者更新应用的过程中，容器停止的条件是啥，会造成服务不可用吗？

##### 答：
经查，重新调度中，应用容器在10秒后被强制关闭。调用链路如下：
client--- xxx.bbbb.citic ---- slb:443 --- ecs:9080 --- container
docker stop --  发 SIGTERM 信号，容器内进程没有管 ---- 10秒后，Docker Daemon 再向 container 发 SIGKILL。

docker stop --time 20 ; Seconds to wait for stop before killing it

perfect的方案：container 中pid为1的进程，能够处理SIGTERM。
```
if s == syscall.SIGTERM {
fmt.Println(“SIGTERM received!”)
//Do something…
}
```

#### 问题10
##### 问：
容器挂载oss卷，进入容器后，查看数据卷报错：
![snap7](https://bjdzliu.oss-cn-beijing.aliyuncs.com/hexo_images/aliyun_QA/aliyun-7)
然后进入/ossdir/lm-activity/lm-wheel 中。输入ls命令。  
再次 ls -l /ossdir/lm-activity/lm-wheel。又正常了。
##### 答：
想知道在oss里创建目录后，挂载oss数据卷的容器会不会同步看到这个目录是么?
默认有缓存，所以可能有延迟的
这个和网络状况有关系，可以参考[ossfs文档](https://github.com/aliyun/ossfs/wiki/FAQ-EN)
