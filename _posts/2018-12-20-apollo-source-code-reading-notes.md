---
layout: post
title: Apollo源码阅读笔记（一）
categories: [Apollo]
description: Apollo源码学习 —— 配置设置到Spring的env中
keywords: apollo, 携程apollo, ctripcorp apollo
---

## Apollo源码阅读笔记（一）

先来一张官方客户端设计图，方便我们了解客户端的整体思路。

![](https://github.com/ctripcorp/apollo/raw/master/doc/images/client-architecture.png)

我们在使用Apollo的时候，需要标记@EnableApolloConfig来告诉程序开启apollo配置，所以这里就以EnableApolloConfig为入口，来看下apollo客户端的实现逻辑。关于apollo的使用方法详见 [这里](https://github.com/ctripcorp/apollo/wiki/Java%E5%AE%A2%E6%88%B7%E7%AB%AF%E4%BD%BF%E7%94%A8%E6%8C%87%E5%8D%97)

### 1.  入口 @EnableApolloConfig 注解

```java
@EnableApolloConfig(value={"application","test-yejg"}) 
```

默认的namespace是application；
通过@EnableApolloConfig注解，引入了ApolloConfigRegistrar

```java
public class ApolloConfigRegistrar implements ImportBeanDefinitionRegistrar {
  @Override
  public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
    AnnotationAttributes attributes = AnnotationAttributes.fromMap(importingClassMetadata.getAnnotationAttributes(EnableApolloConfig.class.getName()));
    String[] namespaces = attributes.getStringArray("value");
    int order = attributes.getNumber("order");
	// 暂存需要关注的namespaces,后面在PropertySourcesProcessor中会把配置属性加载env中
    PropertySourcesProcessor.addNamespaces(Lists.newArrayList(namespaces), order);

    Map<String, Object> propertySourcesPlaceholderPropertyValues = new HashMap<>();
    // to make sure the default PropertySourcesPlaceholderConfigurer's priority is higher than PropertyPlaceholderConfigurer
    propertySourcesPlaceholderPropertyValues.put("order", 0);
    BeanRegistrationUtil.registerBeanDefinitionIfNotExists(registry, PropertySourcesPlaceholderConfigurer.class.getName(), PropertySourcesPlaceholderConfigurer.class, propertySourcesPlaceholderPropertyValues);
	
    BeanRegistrationUtil.registerBeanDefinitionIfNotExists(registry, PropertySourcesProcessor.class.getName(), PropertySourcesProcessor.class);
    BeanRegistrationUtil.registerBeanDefinitionIfNotExists(registry, ApolloAnnotationProcessor.class.getName(), ApolloAnnotationProcessor.class);
    BeanRegistrationUtil.registerBeanDefinitionIfNotExists(registry, SpringValueProcessor.class.getName(), SpringValueProcessor.class);
    BeanRegistrationUtil.registerBeanDefinitionIfNotExists(registry, SpringValueDefinitionProcessor.class.getName(), SpringValueDefinitionProcessor.class);
    BeanRegistrationUtil.registerBeanDefinitionIfNotExists(registry, ApolloJsonValueProcessor.class.getName(), ApolloJsonValueProcessor.class);
  }
}
```
注意上面代码中，通过PropertySourcesProcessor.addNamespaces暂存了namespaces，下面就先沿着 PropertySourcesProcessor来展开



### 2.  配置设置到environment的过程

PropertySourcesProcessor实现了BeanFactoryPostProcessor，并能获取到env

```
public class PropertySourcesProcessor implements BeanFactoryPostProcessor, EnvironmentAware, PriorityOrdered{
    ...
}
```

在Spring应用启动的时候

>   refresh()  –> invokeBeanFactoryPostProcessors(beanFactory) –> PropertySourcesProcessor.postProcessBeanFactory
>
>   —> initializePropertySources();
>
>   —> initializeAutoUpdatePropertiesFeature(beanFactory);

就这样，Apollo的PropertySourcesProcessor就被调用起来了。

在它的postProcessBeanFactory方法中依次调用initializePropertySources和initializeAutoUpdatePropertiesFeature，先来看initializePropertySources做了啥事情：

1.  将NAMESPACE_NAMES （Multimap<Integer, String>）排序；

2.  遍历排序后的namespaces，依次调用 ConfigService.getConfig(namespace) 获取配置信息Config；

3.  将config封装成ConfigPropertySource[Apollo的]，保存到CompositePropertySource[spring-core的]；

    ​	此composite名为 ApolloPropertySources

    ​	ConfigPropertySource继承自spring-core的EnumerablePropertySource<Config>

    ​	代码：composite.addPropertySource(XXX);

4.  循环处理完 NAMESPACE_NAMES 之后，将其清空掉；

5.  将前面循环处理好的compositePropertySource加入到env中；

    ​	加到env时，判断env中是否存在 ApolloBootstrapPropertySources是否存在，确保其在第一的位置，而前面循环处理得到的ApolloPropertySources紧随其后。

    ​	相关代码：

    ​		environment.getPropertySources().addAfter(“XXX source name”, composite);
    ​		environment.getPropertySources().addFirst(composite);	

这部分的逻辑，其实就是佐证了Apollo的[设计思路](https://github.com/ctripcorp/apollo/wiki/Apollo%E9%85%8D%E7%BD%AE%E4%B8%AD%E5%BF%83%E8%AE%BE%E8%AE%A1#31-%E5%92%8Cspring%E9%9B%86%E6%88%90%E7%9A%84%E5%8E%9F%E7%90%86) 。

盗用官方的一张图来简单说明这个流程：

![](https://raw.githubusercontent.com/ctripcorp/apollo/master/doc/images/environment-remote-source.png)

