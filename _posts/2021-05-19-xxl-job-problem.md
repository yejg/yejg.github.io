---
layout: post
title: xxl-job使用遇到的问题（一）
categories: [xxl-job]
description: xxl-job使用遇到的问题
keywords: xxl-job, netty, spring
---

# xxl-job使用遇到的问题（一）



## 1、背景

最近公司在迁移定时任务，以前老的定时任务是基于quartz搭建的分布式集群服务，遇到如下几个瓶颈问题：

- 同一个任务只能有一个节点运行，其他节点不执行，导致性能低，资源也浪费
- 定时任务在抢占执行的时候(数据库锁)，谁先抢到谁执行，导致有些节点忙死，有些节点一直闲置。（没有合理的调度策略）
- 当碰到大量短任务时，各个节点频繁的竞争数据库锁，节点越多这种情况越严重。性能会很低下
- quartz 的分布式仅解决了集群高可用的问题，并没有解决任务分片的问题，不能实现水平扩展（水平扩展的问题，其实通过任务分布式锁的形式也能解决）

反正，最终选定了xxl-job。

想着写写xxl-job迁移说明的，但是网上太多了，官方指导手册也挺多的，还是算了。就记录下遇到的问题+解决方法吧。





## 2、问题

经过一番coding、配置 之后，在xxl-job任务调度中心界面上，点击【执行一次】，调度日志显示失败，错误信息是这样子的
![image](/images/posts/xxl-job/error1.png)<br />

>   xxl-rpc remoting error(Unexpected end of file from server), for url : http://10.25.31.45:9999/run
>

这个ip是我本机的，执行器注册啥的都没问题。



先说解决办法


### 2.1、解决办法

确认下netty的版本是否是4.1.48.Final。

>   Netty3、netty4可以共存，
>
>   但是每个只依赖一个主版本即可，netty4要4.1.48+

检查下pom.xml，果然有猫腻
![image](/images/posts/xxl-job/maven-dependence.png)<br />

正常情况下，xxl-job-core 依赖的netty版本是 4.1.48.Final，在我这里变成了4.0.35~

看了下xxl-job.pom文件里面是使用的 ${netty-all.version} 属性，这说明我的项目里面有别的地方也定义了这个值，覆盖了xxl-job的4.1.48



在项目的pom文件里面定义netty-all.version 为 4.1.48.Final 就搞定了。






### 2.2、原因

mavan在处理依赖的时候，有如下两个原则：
- 最短路径优先
- 相同路径下配置在前面的优先



那另外一个问题，你咋知道就是netty的问题呢？

（先卖个关子，后面再读读xxl-job的源码）