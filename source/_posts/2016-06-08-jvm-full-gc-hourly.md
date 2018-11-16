---
layout: post
title: "JVM每小时执行一次FULL GC问题"
description: tomcat预防内存泄露监听器，会周期性触发FULL GC
date: 2016-06-08 17:26:48 +0800
catalog: true
categories:
- JVM
tags:
- JVM
---

## 引子

最近分析系统中部分机器内存使用率偏高报警问题，发现这部分机器堆内存使用率持续增长，当堆内存接近上限的时候才会触发一次FULL GC；其余机器内存使用率整体波动比较稳定，且FULL GC频率大致是1个小时。  

针对这个报警问题，考虑将这部分机器的JVM FULL GC也调整成1个小时执行一次。开始增加JVM参数`-XX:+UnlockExperimentalVMOptions`，没有起作用。后来才发现这个是由tomcat防止内存泄露的监听器JreMemoryLeakPreventionListener（server.xml中配置）触发的，将tomcat版本由6.0.44降低到了6.0.33就OK了。  

## Tomcat防止内存泄露监听器

tomcat为了防止内存泄露，会注册一个监听器，周期性的触发`System.gc()`，下面是server.xml中监听器的配置  

```
<!-- Prevent memory leaks due to use of particular java/javax APIs-->
<Listener className="org.apache.catalina.core.JreMemoryLeakPreventionListener" />
```

tomcat 7提出了每隔一小时执行一次FULL GC这个[bug](https://bz.apache.org/bugzilla/show_bug.cgi?id=53267)，**Tomcat 7.0.28**版本中将执行频率由一小时修改成 Long.MAX_VALUE-1。并在**Tomcat 6.0.36**版本中同步修复了这个bug。  

**这个bug的修复可以通过下面两个版本修复前和修复后的对比直观的看出来：**  

下面是Tomcat 6.0.33版本中，JreMemoryLeakPreventionListener触发FULL GC的实现（代码为反编译），如下：  

``` java Tomcat 6.0.33版本实现
Class clazz = Class.forName("sun.misc.GC");
Method method = clazz.getDeclaredMethod("requestLatency", new Class[] { Long.TYPE });
method.invoke(null, new Object[] { Long.valueOf(3600000L) });
```
再看Tomcat 6.0.44版本中，修复后的实现：  

``` java Tomcat 6.0.44版本实现
Class clazz = Class.forName("sun.misc.GC");
Method method = clazz.getDeclaredMethod("requestLatency", new Class[] { Long.TYPE });
method.invoke(null, new Object[] { Long.valueOf(9223372036854775806L) });
```
jdk sun.misc.GC类的requestLatency方法，传入的参数表示一次gc请求的延迟时间。会有个低优先级的daemon线程执行调用`System.gc()`

## 防止Tomcat每小时FULL GC

在tomcat没修复此bug之前，可以通过如下方式防止每小时FULL GC：（可阅读[原文](http://mail-archives.apache.org/mod_mbox/tomcat-users/201008.mbox/%3CAANLkTino=BjP5LsBCwncB2HvNDzyKLr5y-8yWdt15a89@mail.gmail.com%3E)）
  
- 增加JVM参数`-XX:+DisableExplicitGC`，这个参数会使显示的调用System.gc()空转，不会执行垃圾回收  
- 不增加JVM参数`-XX:+DisableExplicitGC`，换成增加`-XX:+ExplicitGCInvokesConcurrent`，使FULL GC使用并发垃圾回收器CMS，提高回收效率（CMS并发GC，`stop-the-world`时间较短）  
- 修改server.xml配置，`gcDaemonProtection`参数改为false，默认是true  

``` xml
 <Listener className="org.apache.catalina.core.JreMemoryLeakPreventionListener"
gcDaemonProtection="false"/>
```
- 修改server.xml配置，去掉JreMemoryLeakPreventionListener监听器

> 对于现在，tomcat已经修复了此bug，升级tomcat版本也可以解决每小时执行一次FULL GC问题

## Tomcat每小时GC问题

在网上查这个问题时候，很多人都把每小时执行一次GC作为bug，寻找解决方案（我是反其道而行之，把系统调整成每小时GC一次 -_-）。  

**每小时一次FULL GC到底有何问题？**  
我目前是这么理解的：FULL GC会导致`stop-the-world`，频繁的FULL GC会影响系统的可用性。  
stackoverflow上有个[提问](http://stackoverflow.com/questions/14902928/why-does-the-jvm-of-these-tomcat-servers-perform-a-full-gc-hourly)是这么描述这个问题的：当服务器集群批量启动后，执行FULL GC的频率和时间点大致相同，若FULL GC时间过长，会让负载均衡器觉得各个机器服务不可用，可能导致整个服务下线。  

>但是从FULL GC的角度考虑，对于调整的这个业务系统，目前没有发现FULL GC的`stop-the-world`对系统有严重影响（相对来说，实时系统对GC的低停顿、高吞吐量要求更高），所以我还是把系统改成了每小时执行一次FULL GC的模式。  
>这个问题有待以后深入学习、理解 -_-

## 扩展：jstat分析GC

jdk提供了工具jstat分析系统的GC行为，可以分析minor gc和full gc发生的时间和GC时间  
jstat -gcutil javaPid outputInterval  
jstat -gccause javaPid outputInterval  
**具体的可参考：<http://blog.csdn.net/fenglibing/article/details/6411951>**

