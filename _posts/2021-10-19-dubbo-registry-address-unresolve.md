---
layout: post
title: dubbo注册中心占位符无法解析
categories: [dubbo]
description: dubbo注册中心占位符无法解析
keywords: dubbo, registry, address, spring
---
## dubbo注册中心占位符无法解析问题

### 1、背景

最近搞了2个老项目，想把他们融合到一起。这俩项目情况简介如下：

-   项目一：基于SpringMVC + dubbo，配置读取本地properties文件，少量配置读取apollo
-   项目二：基于Springboot + dubbo，配置读取apollo

本着就高不就低的原则，顺带就把他们拉平到基于springboot的了，没想到这一番操作引入了一个折腾我2天的坑。



### 2、现象

项目合到一起后，处理掉各种报错后，启动，控制台迅速打印出错误日志：

```
Caused by: java.net.UnknownHostException: ${zk.address}
	at java.net.InetAddress.getAllByName0(InetAddress.java:1280)
	at java.net.InetAddress.getAllByName(InetAddress.java:1192)
	at java.net.InetAddress.getAllByName(InetAddress.java:1126)
	at java.net.InetAddress.getByName(InetAddress.java:1076)
	at io.netty.util.internal.SocketUtils$8.run(SocketUtils.java:156)
```

zk.address是写在项目的spring-dubbo-common.xml中，配置内容如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://code.alibabatech.com/schema/dubbo
       http://code.alibabatech.com/schema/dubbo/dubbo.xsd">
    
    <dubbo:application name="dc-dubbo"/>
    <dubbo:registry address="${zk.address}" />
    <dubbo:protocol name="dubbo" port="21660" threadpool="fixed" threads="300"/>
</beans>
```

具体的值是配置在apollo上的。



### 3、问题排查

#### 3.1、apollo配置问题？

首先想到的是，可能是apollo配置问题。就依次做了如下检查：

1.  检查【src/main/resources/META-INF/app.properties】里面的appId是否和apollo配置中心的一致
2.  检查apollo上的配置，是不是真的有zk.address
3.  检查下【C:\opt\data\\ {appId}】目录下的apollo配置缓存，找一下是不是有zk.address的配置

这三点都检查下来，发现配置读取都正常。



#### 3.2、 配置解析问题？

配置都正常，但是占位符却没解析，那么想到解析“模块”出了问题。

解析占位符一般通过Spring的PropertySourcesPlaceholderConfigurer来处理。

在Apollo的源码中，已经注册过PropertySourcesPlaceholderConfigurer这个类了，源码com.ctrip.framework.apollo.spring.spi.DefaultApolloConfigRegistrarHelper摘录如下：

```java
public class DefaultApolloConfigRegistrarHelper implements ApolloConfigRegistrarHelper {

  @Override
  public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
    AnnotationAttributes attributes = AnnotationAttributes
        .fromMap(importingClassMetadata.getAnnotationAttributes(EnableApolloConfig.class.getName()));
    String[] namespaces = attributes.getStringArray("value");
    int order = attributes.getNumber("order");
    PropertySourcesProcessor.addNamespaces(Lists.newArrayList(namespaces), order);

    Map<String, Object> propertySourcesPlaceholderPropertyValues = new HashMap<>();
    propertySourcesPlaceholderPropertyValues.put("order", 0);
      
	// ！！ 这里注册了PropertySourcesPlaceholderConfigurer类 ！！
    BeanRegistrationUtil.registerBeanDefinitionIfNotExists(registry, PropertySourcesPlaceholderConfigurer.class.getName(),
        PropertySourcesPlaceholderConfigurer.class, propertySourcesPlaceholderPropertyValues);
      
    BeanRegistrationUtil.registerBeanDefinitionIfNotExists(registry, PropertySourcesProcessor.class.getName(),
        PropertySourcesProcessor.class);
    BeanRegistrationUtil.registerBeanDefinitionIfNotExists(registry, ApolloAnnotationProcessor.class.getName(),
        ApolloAnnotationProcessor.class);
    BeanRegistrationUtil.registerBeanDefinitionIfNotExists(registry, SpringValueProcessor.class.getName(),
        SpringValueProcessor.class);
    BeanRegistrationUtil.registerBeanDefinitionIfNotExists(registry, SpringValueDefinitionProcessor.class.getName(),
        SpringValueDefinitionProcessor.class);
    BeanRegistrationUtil.registerBeanDefinitionIfNotExists(registry, ApolloJsonValueProcessor.class.getName(),
        ApolloJsonValueProcessor.class);
  }

  @Override
  public int getOrder() {
    return Ordered.LOWEST_PRECEDENCE;
  }
}
```

然后我尝试在PropertySourcesPlaceholderConfigurer里面加断点，看下是不是有进入解析替换配置。
![image](/images/posts/dubbo-registry-address/PropertySourcesPlaceholderConfigurer-breakpoint.png)



从debug来看，配置解析也正常，能正确解析apollo上的配置。

到这里，我有点纳闷了，已经解析到了，咋还提示报错了呢？难道是dubbo的问题？



#### 3.3、dubbo自身的问题？

dubbo的配置，最终会封装成对象。上面配置的zk.address是dubbo:registry的配置，最终会封装成RegistryConfig对象。

类似的还有很多，比如：

-   ApplicationConfig：对应dubbo:application应用配置
-   ProtocolConfig：对应协议配置

于是，我在RegistryConfig#setAddress方法加了个断点。看看对象构造时，传入的address参数是多少。

![image](/images/posts/dubbo-registry-address/RegistryConfig-breakpoint.png)

看来，这个问题还得往前找原因。

当我继续往下执行的时候，idea提示断点进入了3.2节的org.springframework.context.support.PropertySourcesPlaceholderConfigurer#postProcessBeanFactory处。

也就是说，在Spring解析占位符之前，dubbo的对象（RegistryConfig）就开始创建并初始化了。



#### 3.4、对象创建时机问题

从前面的分析下来，问题原因算是找到了：

>   在Spring解析占位符之前，dubbo的对象（RegistryConfig）就开始创建初始化了。导致后面引用的都是未替换占位符的RegistryConfig对象。最终，项目启动才报UnknownHostException: ${zk.address} 异常

再次debug走起，断点停留在RegistryConfig#setAddress方法，看下调用栈：

```
setAddress:118, RegistryConfig (com.alibaba.dubbo.config), RegistryConfig.java
invoke0:-1, NativeMethodAccessorImpl (sun.reflect), NativeMethodAccessorImpl.java
invoke:62, NativeMethodAccessorImpl (sun.reflect), NativeMethodAccessorImpl.java
invoke:43, DelegatingMethodAccessorImpl (sun.reflect), DelegatingMethodAccessorImpl.java
invoke:498, Method (java.lang.reflect), Method.java
setValue:332, BeanWrapperImpl$BeanPropertyHandler (org.springframework.beans), BeanWrapperImpl.java
processLocalProperty:458, AbstractNestablePropertyAccessor (org.springframework.beans), AbstractNestablePropertyAccessor.java
setPropertyValue:278, AbstractNestablePropertyAccessor (org.springframework.beans), AbstractNestablePropertyAccessor.java
setPropertyValue:266, AbstractNestablePropertyAccessor (org.springframework.beans), AbstractNestablePropertyAccessor.java
setPropertyValues:97, AbstractPropertyAccessor (org.springframework.beans), AbstractPropertyAccessor.java
setPropertyValues:77, AbstractPropertyAccessor (org.springframework.beans), AbstractPropertyAccessor.java
applyPropertyValues:1732, AbstractAutowireCapableBeanFactory (org.springframework.beans.factory.support), AbstractAutowireCapableBeanFactory.java
populateBean:1444, AbstractAutowireCapableBeanFactory (org.springframework.beans.factory.support), AbstractAutowireCapableBeanFactory.java
doCreateBean:594, AbstractAutowireCapableBeanFactory (org.springframework.beans.factory.support), AbstractAutowireCapableBeanFactory.java
createBean:517, AbstractAutowireCapableBeanFactory (org.springframework.beans.factory.support), AbstractAutowireCapableBeanFactory.java
lambda$doGetBean$0:323, AbstractBeanFactory (org.springframework.beans.factory.support), AbstractBeanFactory.java
getObject:-1, 1713129148 (org.springframework.beans.factory.support.AbstractBeanFactory$$Lambda$195), Unknown Source
getSingleton:226, DefaultSingletonBeanRegistry (org.springframework.beans.factory.support), DefaultSingletonBeanRegistry.java
doGetBean:321, AbstractBeanFactory (org.springframework.beans.factory.support), AbstractBeanFactory.java
getBean:202, AbstractBeanFactory (org.springframework.beans.factory.support), AbstractBeanFactory.java
getBeansOfType:621, DefaultListableBeanFactory (org.springframework.beans.factory.support), DefaultListableBeanFactory.java
getBeansOfType:1251, AbstractApplicationContext (org.springframework.context.support), AbstractApplicationContext.java
beansOfTypeIncludingAncestors:378, BeanFactoryUtils (org.springframework.beans.factory), BeanFactoryUtils.java
afterPropertiesSet:140, ReferenceBean (com.alibaba.dubbo.config.spring), ReferenceBean.java
invokeInitMethods:1855, AbstractAutowireCapableBeanFactory (org.springframework.beans.factory.support), AbstractAutowireCapableBeanFactory.java
initializeBean:1792, AbstractAutowireCapableBeanFactory (org.springframework.beans.factory.support), AbstractAutowireCapableBeanFactory.java
doCreateBean:595, AbstractAutowireCapableBeanFactory (org.springframework.beans.factory.support), AbstractAutowireCapableBeanFactory.java
createBean:517, AbstractAutowireCapableBeanFactory (org.springframework.beans.factory.support), AbstractAutowireCapableBeanFactory.java
lambda$doGetBean$0:323, AbstractBeanFactory (org.springframework.beans.factory.support), AbstractBeanFactory.java
getObject:-1, 1713129148 (org.springframework.beans.factory.support.AbstractBeanFactory$$Lambda$195), Unknown Source
getSingleton:226, DefaultSingletonBeanRegistry (org.springframework.beans.factory.support), DefaultSingletonBeanRegistry.java
doGetBean:321, AbstractBeanFactory (org.springframework.beans.factory.support), AbstractBeanFactory.java
getTypeForFactoryBean:1646, AbstractBeanFactory (org.springframework.beans.factory.support), AbstractBeanFactory.java
getTypeForFactoryBean:895, AbstractAutowireCapableBeanFactory (org.springframework.beans.factory.support), AbstractAutowireCapableBeanFactory.java
isTypeMatch:613, AbstractBeanFactory (org.springframework.beans.factory.support), AbstractBeanFactory.java
doGetBeanNamesForType:537, DefaultListableBeanFactory (org.springframework.beans.factory.support), DefaultListableBeanFactory.java
getBeanNamesForType:495, DefaultListableBeanFactory (org.springframework.beans.factory.support), DefaultListableBeanFactory.java
getBeansOfType:617, DefaultListableBeanFactory (org.springframework.beans.factory.support), DefaultListableBeanFactory.java
getBeansOfType:609, DefaultListableBeanFactory (org.springframework.beans.factory.support), DefaultListableBeanFactory.java
getBeansOfType:1243, AbstractApplicationContext (org.springframework.context.support), AbstractApplicationContext.java
processPropertyPlaceHolders:367, MapperScannerConfigurer (org.mybatis.spring.mapper), MapperScannerConfigurer.java
postProcessBeanDefinitionRegistry:338, MapperScannerConfigurer (org.mybatis.spring.mapper), MapperScannerConfigurer.java
invokeBeanDefinitionRegistryPostProcessors:280, PostProcessorRegistrationDelegate (org.springframework.context.support), PostProcessorRegistrationDelegate.java
invokeBeanFactoryPostProcessors:126, PostProcessorRegistrationDelegate (org.springframework.context.support), PostProcessorRegistrationDelegate.java
invokeBeanFactoryPostProcessors:707, AbstractApplicationContext (org.springframework.context.support), AbstractApplicationContext.java
refresh:533, AbstractApplicationContext (org.springframework.context.support), AbstractApplicationContext.java
refresh:143, ServletWebServerApplicationContext (org.springframework.boot.web.servlet.context), ServletWebServerApplicationContext.java
refresh:758, SpringApplication (org.springframework.boot), SpringApplication.java
refresh:750, SpringApplication (org.springframework.boot), SpringApplication.java
refreshContext:397, SpringApplication (org.springframework.boot), SpringApplication.java
run:315, SpringApplication (org.springframework.boot), SpringApplication.java
main:36, DcRemoteApplication (com.yc.dc.remote), DcRemoteApplication.java
```

从调用栈，发现一点奇怪的调用路径：

>   在org.mybatis.spring.mapper.MapperScannerConfigurer#processPropertyPlaceHolders方法中，调用了applicationContext.getBeansOfType(PropertyResourceConfigurer.class); 从而引发了后面的一些列的对象创建在PropertyResourceConfigurer之前。

```java
public class MapperScannerConfigurer implements BeanDefinitionRegistryPostProcessor, InitializingBean, ApplicationContextAware, BeanNameAware {
  @Override
  public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
    if (this.processPropertyPlaceHolders) { // true
      processPropertyPlaceHolders();
    }
    ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);
    scanner.setAddToConfig(this.addToConfig);
    scanner.setAnnotationClass(this.annotationClass);
    scanner.setMarkerInterface(this.markerInterface);
    scanner.setSqlSessionFactory(this.sqlSessionFactory);
    scanner.setSqlSessionTemplate(this.sqlSessionTemplate);
    scanner.setSqlSessionFactoryBeanName(this.sqlSessionFactoryBeanName);
    scanner.setSqlSessionTemplateBeanName(this.sqlSessionTemplateBeanName);
    scanner.setResourceLoader(this.applicationContext);
    scanner.setBeanNameGenerator(this.nameGenerator);
    scanner.setMapperFactoryBeanClass(this.mapperFactoryBeanClass);
    if (StringUtils.hasText(lazyInitialization)) {
      scanner.setLazyInitialization(Boolean.valueOf(lazyInitialization));
    }
    scanner.registerFilters();
    scanner.scan(
        StringUtils.tokenizeToStringArray(this.basePackage, ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS));  
  }
  
  private void processPropertyPlaceHolders() {
    // 这里调 getBeansOfType(PropertyResourceConfigurer.class)
    Map<String, PropertyResourceConfigurer> prcs = applicationContext.getBeansOfType(PropertyResourceConfigurer.class);
    // ...此处省略若干代码...  
  }
  
  // ...此处省略若干代码...
}
```

【applicationContext.getBeansOfType(PropertyResourceConfigurer.class);】执行过程中

>   Spring一路执行到org.springframework.beans.factory.support.DefaultListableBeanFactory#doGetBeanNamesForType，
>
>   并在方法内，循环所有beanDefinitionNames，逐一判断beanDefinition是不是PropertyResourceConfigurer类型的（isTypeMatch方法）
>
>   判断的时候，就需要从Spring容器中取这个bean，取着取着发现没有，就开始doCeateBean-->invokeInitMethods-->afterPropertiesSet 了。

在创建dubbo消费者的时候，消费者接口被封装成ReferenceBean，在其afterPropertiesSet方法中，依赖RegistryConfig对象，从而依赖注册中心配置的地址。

而这些，都是发生在PropertyResourceConfigurer创建之前。

>   **此处特别说明，我项目里面使用的mybatis-spring是2.0.3版本**



### 4、 解决方法

#### 4.1、方法一

前面分析了，MapperScannerConfigurer对象的boolean属性processPropertyPlaceHolders为true，导致执行了processPropertyPlaceHolders()方法。

那么，据此可以自定义一个MapperScannerConfigurer对象，设置processPropertyPlaceHolders为false。

```java
// 去掉 @MapperScan 注解，通过下面的方式设置MapperScannerConfigurer
@Bean
public MapperScannerConfigurer dsCrmMapperScannerConfigurer() {
	MapperScannerConfigurer configurer = new MapperScannerConfigurer();
	configurer.setProcessPropertyPlaceHolders(false);
	configurer.setBasePackage("com.yc.dc.mapper");
	configurer.setSqlSessionFactoryBeanName("dsCrmSqlSessionFactory");
	return configurer;
}
```

通过这种方式试了下，发现果然可行。



**为啥可行？** 究其原因如下：

MapperScannerConfigurer一般通过MapperScannerRegistrar引入进来。

mybatis-spring 是从2.0.2版本开始，引入MapperScannerConfigurer的时候，设置了processPropertyPlaceHolders值为true。

才会在扫描mapper之前，调了一下 processPropertyPlaceHolders()方法。

```java
// org.mybatis.spring.annotation.MapperScannerRegistrar，mybatis-spring 2.0.2及以上版本
void registerBeanDefinitions(AnnotationAttributes annoAttrs, BeanDefinitionRegistry registry, String beanName) {

    BeanDefinitionBuilder builder = BeanDefinitionBuilder.genericBeanDefinition(MapperScannerConfigurer.class);
    builder.addPropertyValue("processPropertyPlaceHolders", true); // 看这里！！

    Class<? extends Annotation> annotationClass = annoAttrs.getClass("annotationClass");
    if (!Annotation.class.equals(annotationClass)) {
      builder.addPropertyValue("annotationClass", annotationClass);
    }
    // .....
}


// org.mybatis.spring.annotation.MapperScannerRegistrar，mybatis-spring 2.0.1及以下版本
void registerBeanDefinitions(AnnotationAttributes annoAttrs, BeanDefinitionRegistry registry) {
    ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);
    // ClassPathMapperScanner里面就没有getBeansOfType(PropertyResourceConfigurer.class)
    Optional.ofNullable(resourceLoader).ifPresent(scanner::setResourceLoader);
    Class<? extends Annotation> annotationClass = annoAttrs.getClass("annotationClass");
    if (!Annotation.class.equals(annotationClass)) {
      scanner.setAnnotationClass(annotationClass);
    }
	// .....
}
```



#### 4.2、方法二

降低mybatis-spring的版本到 2.0.1 或者 更低版本。



### 5、未完待续

到这里，问题就查清楚了，并且解决办法也有了。但真的结束了吗？
木有，合并前的老项目二使用的就是mybatis-spring-2.0.3.jar，但是他没有报【UnknownHostException: ${zk.address}】，也没有自定义MapperScannerConfigurer设置processPropertyPlaceHolders为false
那为什么？
下篇文章分析~




















