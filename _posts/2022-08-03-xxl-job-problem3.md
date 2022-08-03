---
layout: post
title: xxl-job使用遇到的问题（三）
categories: [xxl-job]
description: xxl-job使用遇到的问题
keywords: xxl-job, netty, Connection reset by peer
---

# xxl-job使用遇到的问题（三）



## 1、问题现象

这两天系统日志里面总会出现xxl-job的报错，但是job执行器能被正常调用并执行，调度日志也都显示成功。

不影响正常使用，但是error报错多了，就容易触发告警。o(╯□╰)o

>   content: >>>>>>>>>>> xxl-job provider netty_http server caught exception
>
>   java.io.IOException: Connection reset by peer
>
>   ​	at sun.nio.ch.FileDispatcherImpl.read0(Native Method) ~[?:1.8.0_65]
>
>   ​	at sun.nio.ch.SocketDispatcher.read(SocketDispatcher.java:39) ~[?:1.8.0_65]
>
>   ​	at sun.nio.ch.IOUtil.readIntoNativeBuffer(IOUtil.java:223) ~[?:1.8.0_65]
>
>   ​	at sun.nio.ch.IOUtil.read(IOUtil.java:192) ~[?:1.8.0_65]
>
>   ​	at sun.nio.ch.SocketChannelImpl.read(SocketChannelImpl.java:380) ~[?:1.8.0_65]
>
>   ​	at io.netty.buffer.PooledByteBuf.setBytes(PooledByteBuf.java:253) ~[netty-all-4.1.48.Final.jar:4.1.48.Final]
>
>   ​	at io.netty.buffer.AbstractByteBuf.writeBytes(AbstractByteBuf.java:1133) ~[netty-all-4.1.48.Final.jar:4.1.48.Final]
>
>   ​	at io.netty.channel.socket.nio.NioSocketChannel.doReadBytes(NioSocketChannel.java:350) ~[netty-all-4.1.48.Final.jar:4.1.48.Final]
>
>   ​	at io.netty.channel.nio.AbstractNioByteChannel$NioByteUnsafe.read(AbstractNioByteChannel.java:148) [netty-all-4.1.48.Final.jar:4.1.48.Final]
>
>   ​	at io.netty.channel.nio.NioEventLoop.processSelectedKey(NioEventLoop.java:714) [netty-all-4.1.48.Final.jar:4.1.48.Final]
>
>   ​	at io.netty.channel.nio.NioEventLoop.processSelectedKeysOptimized(NioEventLoop.java:650) [netty-all-4.1.48.Final.jar:4.1.48.Final]
>
>   ​	at io.netty.channel.nio.NioEventLoop.processSelectedKeys(NioEventLoop.java:576) [netty-all-4.1.48.Final.jar:4.1.48.Final]
>
>   ​	at io.netty.channel.nio.NioEventLoop.run(NioEventLoop.java:493) [netty-all-4.1.48.Final.jar:4.1.48.Final]
>
>   ​	at io.netty.util.concurrent.SingleThreadEventExecutor$4.run(SingleThreadEventExecutor.java:989) [netty-all-4.1.48.Final.jar:4.1.48.Final]
>
>   ​	at io.netty.util.internal.ThreadExecutorMap$2.run(ThreadExecutorMap.java:74) [netty-all-4.1.48.Final.jar:4.1.48.Final]
>
>   ​	at io.netty.util.concurrent.FastThreadLocalRunnable.run(FastThreadLocalRunnable.java:30) [netty-all-4.1.48.Final.jar:4.1.48.Final]
>
>   ​	at java.lang.Thread.run(Thread.java:745) [?:1.8.0_65]
>
>   level: ERROR
>
>   method: com.xxl.job.core.server.EmbedServer$EmbedHttpServerHandler-exceptionCaught:238





## 2、原因及其解决办法

主要是最近换了阿里云的SLB服务，云提供商lb的健康检查引发此问题。

执行器的9999端口，分别在阿里云和腾讯云配置了lb，而且是tcp协议的。而xxljob调度中心与执行器的心跳底层是依赖netty进行的http协议维持。

解决办法如下：

-   9999端口在lb上配置http协议；

-   关闭lb心跳检查；

-   自定义tcp协议；

-   关闭lb；

-   终极方案 - 关闭 EmbedServer 的日志打印

    logging.level.com.xxl.job.core.server.EmbedServer = OFF

阿里云SLB监听机制：

![](/images/posts/xxl-job/ali-slb.png)







## 3、参考资料

[xxl-job issue1088](https://github.com/xuxueli/xxl-job/issues/1088)

[阿里云SLB](https://help.aliyun.com/document_detail/171710.htm)















