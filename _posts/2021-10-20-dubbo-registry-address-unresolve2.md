---
layout: post
title: dubbo注册中心占位符无法解析（二）
categories: [dubbo]
description: dubbo注册中心占位符无法解析
keywords: dubbo, registry, address, spring
---
## dubbo注册中心占位符无法解析问题

[前面](/2021/10/19/dubbo-registry-address-unresolve)分析了dubbo注册中心占位符无法解析的问题。

并给出了2种解决办法：

-   降低mybatis-spring的版本至2.0.1及以下
-   自定义MapperScannerConfigurer，设置processPropertyPlaceHolders为false

不过，末尾遗留了一个疑问，这篇文章就来分析下这个问题

>   合并前的老项目二使用的就是mybatis-spring-2.0.3.jar，但是他没有报【UnknownHostException: ${zk.address}】，也没有自定义MapperScannerConfigurer设置processPropertyPlaceHolders为false



### 1、问题分析

根据前面的经验，在如下位置加上断点，debug走起

```java
org.springframework.context.support.PropertySourcesPlaceholderConfigurer#postProcessBeanFactory
com.alibaba.dubbo.config.RegistryConfig#setAddress
```

一番debug走下来，发现RegistryConfig#setAddress确实传的是apollo上配置的值。

也就是说PropertySourcesPlaceholderConfigurer在RegistryConfig对象初始化之前就调了。

这不打脸了吗？前面一顿分析，解决方法都给出来了，￣□￣｜｜

继续往前分析下，看看调用栈，RegistryConfig是因为ReferenceBean#afterPropertiesSet的执行，而被创建的。那么，再增加一个断点

```java
com.alibaba.dubbo.config.spring.ReferenceBean#afterPropertiesSet
```

**多次**debug发现，进入ReferenceBean#afterPropertiesSet的时候，在如下位置其实是报错了

```java
public void afterPropertiesSet() throws Exception {
        // ...此处省略若干代码...  
        if (getApplication() == null
                && (getConsumer() == null || getConsumer().getApplication() == null)) {
            Map<String, ApplicationConfig> applicationConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, ApplicationConfig.class, false, false);
           // ！！ BeanFactoryUtils.beansOfTypeIncludingAncestors报错了 ！！
            
            if (applicationConfigMap != null && applicationConfigMap.size() > 0) {
            // ...此处省略若干代码... 
            }
            // ...此处省略若干代码... 
        }
}
```

![image](/images/posts/dubbo-registry-address/BeanCreationException.png)

完整的错误消息

>   org.springframework.beans.factory.BeanCreationException: Error creating bean with name '${dubbo.application.name}': Error setting property values; nested exception is org.springframework.beans.PropertyBatchUpdateException; nested PropertyAccessExceptions (1) are:
>   PropertyAccessException 1: org.springframework.beans.MethodInvocationException: Property 'name' threw exception; nested exception is java.lang.IllegalStateException: Invalid name="${dubbo.application.name}" contains illegal character, only digit, letter, '-', '_' or '.' is legal.

这不过这个错误，Spring最终没有抛出来，不仅没抛出异常，还destroy了这个创建了一半的bean

![image](/images/posts/dubbo-registry-address/DealBeanCreationException.png)

并在后面，打印了毫不起眼的debug级别日志

>   DEBUG o.s.b.f.s.DefaultListableBeanFactory - Bean creation exception on non-lazy FactoryBean type check: org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'jobCallback': Invocation of init method failed; nested exception is org.springframework.beans.factory.BeanCreationException: Error creating bean with name '${dubbo.application.name}': Error setting property values; nested exception is org.springframework.beans.PropertyBatchUpdateException; nested PropertyAccessExceptions (1) are:
>   PropertyAccessException 1: org.springframework.beans.MethodInvocationException: Property 'name' threw exception; nested exception is java.lang.IllegalStateException: Invalid name="${dubbo.application.name}" contains illegal character, only digit, letter, '-', '_' or '.' is legal.

原来还有这种骚操作！

>   MapperScannerConfigurer --> 创建 -->  未解析dubbo.application.name，创建ApplicationConfig异常  -->  销毁掉 延后再创建 --> PropertySourcesPlaceholderConfigurer  --> 延后创建



再回过头来看了下 老项目二 的spring-dubbo-common.xml里面的配置，它在配置的时候 用的${dubbo.application.name}，而不是具体值！

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://code.alibabatech.com/schema/dubbo
       http://code.alibabatech.com/schema/dubbo/dubbo.xsd">
    
    <dubbo:application name="${dubbo.application.name}"/>
    <!--  
		新项目里面，我写的是这样子的
		<dubbo:application name="dc-dubbo"/>
    -->
    
    <dubbo:registry address="${zk.address}" protocol="zookeeper"/>
    <dubbo:protocol name="dubbo" port="21660" threadpool="fixed" threads="300"/>
</beans>
```

啊！！我有一种负负得正、错错得对的感觉！！



### 2、解决办法

#### 2.1、方法三

（[续前面的方法一、方法二](/2021/10/19/dubbo-registry-address-unresolve)）

xml改成 

```xml
<dubbo:application name="${dubbo.application.name}"/>
```

并增加dubbo.application.name的配置




















