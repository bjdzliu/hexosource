---
title: 分析Metaspace不停增长的原因
date: 2021-3-12 07:42:47
tags:
- linux
categories:
- Technical Notes
---


### 现象：
从监控软件上查看jvm堆区使用
![edenspace曲线](
https://bjdzliu.oss-cn-beijing.aliyuncs.com/hexo_images/metaspace/edenspace.png)


非堆区，
![metaspace曲线](
https://bjdzliu.oss-cn-beijing.aliyuncs.com/hexo_images/metaspace/metaspace_usage.png)



metaspace空间持续上升后，metaspace+edenspace > 7G, 超过了 pod request limit，发生oom，容器被os kill

pod重新拉起一个新的容器



### 概念：
![JVM-1.8 内存模型](https://bjdzliu.oss-cn-beijing.aliyuncs.com/hexo_images/metaspace/jvm-pic.png)


Metaspace 直接在OS上分配的内存。

Metaspace 存放的是加载的class

加载class的过程是由class loader完成。

classloader：
[des in oracle](https://docs.oracle.com/javase/8/docs/api/java/lang/ClassLoader.html)

类加载器尝试定位或生成构成类定义的数据,比如从文件系统获得class文件。  

或者动过动态编译并加载


### 分析：
检查当前jvm中的类加载器及加载的类，查询classlloader中加载的类数量和列表

步骤参阅  [Arthas](http://arthas.gitee.io/)

查询结果如下：
```

[arthas@1]$ classloader
name                                                                                                               numberOfInstances                        loadedCountTotal
org.apache.catalina.loader.ParallelWebappClassLoader                                          1                                                 13431
org.drools.core.rule.JavaDialectRuntimeData$PackageClassLoader                       172                                             8494
BootstrapClassLoader                                                                                               1                                               6083
org.drools.core.base.ClassFieldAccessorCache$DefaultByteArrayClassLoader        169                                            4733
```



```
[arthas@1]$ classloader -a --org.apache.catalina.loader.ParallelWebappClassLoader > class.list

```

统计 class.list 中类 差距比较大的如下  

| class name      | prod | non-prod |
| ----------- | ----------- | ----------- |
| com.citic.user.assignment.checkiint.ruleengine.xxxxx      | 17338 | 8482 |



### 可能的原因：
大量动态加载生成的class



### 建议的解决方案：
1）通过设置MaxMetaspaceSize，触发metaspace中的fullgc，清理没有具体引用的class。
结果: 经验证，fullgc已经将整个java进程冻结，服务请求失败。

因为，默认MaxMetaspaceSize没有设置，则无限大，意味着metaspace没有上限。

(

In JDK 8, the permanent generation was removed and the class metadata is allocated in native memory. The amount of native memory that can be used for class metadata is by default unlimited. Use the option MaxMetaspaceSize to put an upper limit on the amount of native memory used for class metadata.
[摘自 jdk8 doc](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/considerations.html#:~:text=The%20default%20size%20of%20MetaspaceSize%20is%20platform-dependent%20and,%22Typical%20Heap%20Printout%22.%20Example%2011-1%20Typical%20Heap%20Printout).

)

2) 检查代码生成这些类的方式，一般为静态代理或动态代理。



背景知识：
java里的class文件加载：

方式1：根据class文件，读取到内存

方式2：获取一个引用，把这个引用的对象的class，加载到内存。这是反射的实现方式

动态加载利用了反射原理，jdk中提供了默认的类库、或者加载第三方类库来实现动态加载。

默认类库：JDK Proxy动态代理，api在包java.lang.reflect中

在代码中，以下语句(fake code)，会触发class加载，该class是某个对象的class：
```
class DynamicProxyHandler implements InvocationHandler {
...
return Proxy.newProxyInstance(target.getClass().getClassLoader(), target.getClass().getInterfaces(), this);
...}
```
target.getClass() 获得被代理的对象的类，该类加载到jvm中。
