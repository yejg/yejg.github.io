---
layout: post
title: xxl-job使用遇到的问题（二）
categories: [xxl-job]
description: xxl-job使用遇到的问题
keywords: xxl-job, netty, spring
---

# xxl-job使用遇到的问题（二）



## 1、问题现象

最近有个老定时任务迁移到xxl-job的时候，遇到一个小问题。虽然很快解决，但是还是有必要记录一下~

job迁移的时候，在执行方法上标记@XxlJob("test")，然后在管理控制台上，添加任务，点击执行一次的时候，调度日志提示

>   \>>>>>>>>>>>触发调度<<<<<<<<<<<
>   触发调度：
>   address：http://10.25.31.45:9999/
>   code：500
>   msg：job handler [test] not found.

检查了下代码，没啥问题。注解加了，XxlJobSpringExecutor也有，properties配置也ok，为啥提示找不到呢？



大致代码也贴一下

```java
public interface ITestJob {
    void method1();
}


@Service
public class TestJobImpl implements ITestJob {

    @XxlJob("test")
    public ReturnT<String> demoJobHandler() throws Exception {
        // job业务逻辑【历史代码】
        return ReturnT.SUCCESS;
    }

    // 历史代码方法
    @Override
    @Async
    public void method1() {
        // 历史代码逻辑
    }
}
```

其实就是在历史代码的基础上，改了下方法返回值，加了个@XxlJob("test")注解。其他没动



## 2、排查

既然提示找不到job handler，那问题肯定是出在客户端了。

前面阅读源码的时候，已经看过，@XxlJob注解的解析是在 XxlJobSpringExecutor类里面。快读看了下这个类，重点看了下initJobHandlerMethodRepository方法

```java
private void initJobHandlerMethodRepository(ApplicationContext applicationContext) {
        if (applicationContext == null) {
            return;
        }
        // init job handler from method
        String[] beanDefinitionNames = applicationContext.getBeanNamesForType(Object.class, false, true);
        for (String beanDefinitionName : beanDefinitionNames) {
            Object bean = applicationContext.getBean(beanDefinitionName);

            Map<Method, XxlJob> annotatedMethods = null;   // referred to ：org.springframework.context.event.EventListenerMethodProcessor.processBean
            try {
                annotatedMethods = MethodIntrospector.selectMethods(bean.getClass(),
                        new MethodIntrospector.MetadataLookup<XxlJob>() {
                            @Override
                            public XxlJob inspect(Method method) {
                                return AnnotatedElementUtils.findMergedAnnotation(method, XxlJob.class);
                            }
                        });
            } catch (Throwable ex) {
                logger.error("xxl-job method-jobhandler resolve error for bean[" + beanDefinitionName + "].", ex);
            }
            if (annotatedMethods==null || annotatedMethods.isEmpty()) {
                continue;
            }
 
            
            
            ......
            
 }
```

在这块加个断点看看，果然拿到的   annotatedMethods 是空的，难怪提示找不到呢！



### 3、原因

上面的demoJobHandler方法头上标记了注解的，为什么annotatedMethods是空的呢？

这是因为下面这行代码取到的bean，不是TestJobImpl这个类

>   Object bean = applicationContext.getBean(beanDefinitionName);

而是一个代理类，并且是jdk的动态代理的类。

jdk动态代理，啥特性？

嗯，基于接口的~~，接口里面只申明了一个方法 method1，那【MethodIntrospector.selectMethods】肯定找不到有XxlJob注解的方法了！



#### 3.1、 解决办法

知道了这一点，那就好解决了。

可供参考的解决办法：

方法一、把job这个方法单独拎出来

方法二、去掉接口，让Spring使用cglib的代理。这样取到的代理类就包含2个方法了，就可以找到有注解的那个方法了

方法三、如果上面2个都不想改，那就在接口里面申明一个demoJobHandler方法，并且在接口方法申明上标记@XxlJob("test")注解



















