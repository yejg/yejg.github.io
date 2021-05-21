---
layout: post
title: xxl-job源码阅读一（客户端）
categories: [xxl-job]
description: xxl-job客户端源码(2.3.0)
keywords: xxl-job, netty, spring
---

# xxl-job源码阅读一（客户端）




## 1、源码入口

使用xxl-job的时候，需要引入一个jar，然后还需要往Spring容器注入XxlJobSpringExecutor

```java
<dependency>
	<groupId>com.xuxueli</groupId>
	<artifactId>xxl-job-core</artifactId>
	<version>2.3.0</version>
</dependency>

	@Bean
    public XxlJobSpringExecutor xxlJobExecutor() {
        XxlJobSpringExecutor xxlJobSpringExecutor = new XxlJobSpringExecutor();
        xxlJobSpringExecutor.setAdminAddresses(adminAddresses);
        xxlJobSpringExecutor.setAppname(appname);
        xxlJobSpringExecutor.setAddress(address);
        xxlJobSpringExecutor.setIp(ip);
        xxlJobSpringExecutor.setPort(port);
        xxlJobSpringExecutor.setAccessToken(accessToken);
        xxlJobSpringExecutor.setLogPath(logPath);
        xxlJobSpringExecutor.setLogRetentionDays(logRetentionDays);
        return xxlJobSpringExecutor;
    }
```

我们就可以顺着这个XxlJobSpringExecutor，分析下这个xxl-job-core做了些什么。  





## 2、执行器启动

XxlJobSpringExecutor代码比较简洁，大致框架如下：

```java
public class XxlJobSpringExecutor extends XxlJobExecutor implements ApplicationContextAware, SmartInitializingSingleton, DisposableBean {
    
	@Override
    public void afterSingletonsInstantiated() {
        // init JobHandler Repository (for method)
        initJobHandlerMethodRepository(applicationContext);

        // refresh GlueFactory
        GlueFactory.refreshInstance(1);

        // super start
        try {
            super.start();
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
    
  	@Override
    public void destroy() {
        super.destroy();
    }  
    
    // ......

}
```

1、实现了SmartInitializingSingleton接口，在项目启动的时候，会调到afterSingletonsInstantiated方法。这个方法又可以作为进一步阅读的下个入口了；

2、实现了DisposableBean接口，在系统停止的时候，调用destroy方法。

3、实现ApplicationContextAware，只是为了获取applicationContext对象，这个没啥好说的。



先来看看初始化


### 2.1、解析Xxl-job注解

在使用xxl-job的时候，我们会写@XxlJob注解，来告诉xxl-job这是个定时任务的方法。

那么xxl-job就得解析这个注解并做一系列处理。对吧？

这个事情，就是在afterSingletonsInstantiated方法里面调initJobHandlerMethodRepository去做的。代码略长，就不贴出来了。具体实现逻辑如下：

1.  通过applicationContext.getBeanNamesForType获取全部bean

2.  遍历所有bean，找到有@XxlJob标记的方法，并判断@XxlJob标记的name值是否有重复，如果重复则报错

3.  如果有配置init和destroy方法，则通过反射找到他们

4.  将信息组装成MethodJobHandler对象，保存到ConcurrentHashMap中

    >   registJobHandler(name, new MethodJobHandler(bean, executeMethod, initMethod, destroyMethod));




### 2.2、处理glue模式

一般基于Spring使用的时候，都是bean模式，即

>   任务以JobHandler方式维护在执行器端；需要结合 "JobHandler" 属性匹配执行器中任务；

而glue模式是指

>   任务以源码方式维护在调度中心；

这里初始化了一个SpringGlueFactory



### 2.3、启动

这里调用父类XxlJobExecutor的start方法。方法主要执行下面几个大的步骤：

```java
public class XxlJobExecutor  {
	...
    public void start() throws Exception {

        // init logpath
        XxlJobFileAppender.initLogPath(logPath);

        // init invoker, admin-client
        initAdminBizList(adminAddresses, accessToken);


        // init JobLogFileCleanThread
        JobLogFileCleanThread.getInstance().start(logRetentionDays);

        // init TriggerCallbackThread
        TriggerCallbackThread.getInstance().start();

        // init executor-server
        initEmbedServer(address, ip, port, appname, accessToken);
    }
    ...
}
```

1.  初始化执行器日志路径，默认 /data/applogs/xxl-job/jobhandler 。

    这个XxlJobFileAppender是个单独写日志文件的工具类。在xxl-job-admin界面上，可以通过界面查看定时任务调度执行的日志。我们在业务代码中，也可以通过XxlJobHelper.log方法，写自己的日志（老版本是XxlJobLogger.log）

2.  根据配置的adminAddresses地址，构造AdminBiz列表（后面注册、调服务端接口等，会需要调到这个地址） 

3.  启动一个daemon线程，每天定期清理调度日志文件（上述1步骤目录下的文件）

4.  定义一个LinkedBlockingQueue，这个queue里面放job执行结果。然后启动triggerCallbackThread和triggerRetryCallbackThread 两个线程，向job-admin反馈job执行结果。

    >   这里为啥是2个线程去给admin端反馈执行结果呢？
    >
    >   原来，正常情况下，只有triggerCallbackThread从queue里面拿数据，提交到admin。
    >
    >   但是当它提交失败的时候，triggerCallbackThread就会写一个callbacklog文件。再由triggerRetryCallbackThread读取callbacklog文件，并向admin提交执行结果。

5.  构造EmbedServer并启动

    构造EmbedServer需要url，这里涉及到三个配置。

    ```properties
    ### 执行器注册 [选填]：优先使用该配置作为注册地址，为空时使用内嵌服务 ”IP:PORT“ 作为注册地址。从而更灵活的支持容器类型执行器动态IP和动态映射端口问题。
    xxl.job.executor.address=
    ### 执行器IP [选填]：默认为空表示自动获取IP，多网卡时可手动设置指定IP，该IP不会绑定Host仅作为通讯实用；地址信息用于 "执行器注册" 和 "调度中心请求并触发任务"；
    xxl.job.executor.ip=
    ### 执行器端口号 [选填]：小于等于0则自动获取；默认端口为9999，单机部署多个执行器时，注意要配置不同执行器端口；
    xxl.job.executor.port=9999
    ```

    EmbedServer是基于netty实现的一个服务，监听9999端口，并主要响应xxl-job-admin的调度请求。从EmbedHttpServerHandler中可以看出，admin调度中心往执行器发的请求，主要有以下5个：

    ```
    beat：调度中心检测执行器是否在线时使用
    idleBeat：调度中心检测 指定执行器 上 指定任务 是否忙碌（运行中）时使用 （注意这里2个“指定”）
    run：触发任务执行
    kill：终止任务
    	这里是根据jobId找到对应的jobThread，再改变执行标记后，调interrupt方法。
    	（所以如果你想优雅地停止一个线程，也可以通过线程标记+interrupt方式）
    log：查看执行日志
    ```

    执行器什么时候往调用中心注册的呢？

    答案是：在EmbedServer的start方法中，启动了一个thread，在其内部调了【startRegistry(appname, address);】

    而这行代码里面，又启动了一个registryThread，每30秒注册当前执行器。



至此，xxl-job客户端的逻辑大致分析清楚了，下一节再看看admin的代码

