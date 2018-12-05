---
layout: post
title: 向Spring容器中注册组件的方法
categories: [Spring]
description: 向Spring容器中注册组件的方法汇总小结
keywords: Spring, 容器, 注册
---

#### 向Spring容器中注册组件的方法

##### 1、通过xml定义

```xml
<bean class="">
    <property name="" value=""></property>
</bean>
```
##### 2、通过注解

  这种方式比较常见，通常用@Controller、@Component、@Service等等

##### 3、通过@Bean注解

  比如下面的代码往容器中注册一个Person对象

```java
@Bean
public Person person(){
    return new Person("张三", 20);
}
```

默认情况下，使用方法【person()】名person作为Person对象的注册id  
也可以通过修改方法名或者使用@Bean(“customBeanName”)

##### 4、通过实现FactoryBean<T>接口

```java
public interface FactoryBean<T> {
    T getObject() throws Exception;
    Class<?> getObjectType();
    boolean isSingleton();
}

// Sample
public class PersonFactoryBean implements FactoryBean<Person> {
    ....
}
```

实现上述接口的3个方法，并把PersonFactoryBean注册到容器中，就可以把Person也注册到容器中。
具体创建Person过程的源码可以看FactoryBeanRegistrySupport#getObjectFromFactoryBean方法。

```java
// 如下代码拿到的是Person对象
applicationContext.getBean("personFactoryBean")

// 如果想要拿到PersonFactoryBean对象，可以再前面加&
applicationContext.getBean("&personFactoryBean")
```

##### 5、通过@Import注解

先来看看源码：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Import {
    /**
    * {@link Configuration}, {@link ImportSelector}, {@link ImportBeanDefinitionRegistrar}
    * or regular component classes to import.
    */
    Class<?>[] value();
}
```

源码注释写的也很清楚，可以引入 配置类、ImportSelector、ImportBeanDefinitionRegistrar，甚至是普通class。 通过@Import，我们可以使用如下方式注册组件：

```java
@Import({Person.class, MyImportSelector.class,MyImportBeanDefinitionRegistrar.class})
```

其中：

-   MyImportSelector实现了ImportSelector接口，selectImports方法返回类全名的String[]都会被注册到容器中

-   MyImportBeanDefinitionRegistrar实现了ImportBeanDefinitionRegistrar接口

    ```java
// Sample
public class MyImportBeanDefinitionRegistrar implements ImportBeanDefinitionRegistrar {
    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        // 指定Bean定义信息
        RootBeanDefinition beanDefinition = new RootBeanDefinition(Person.class);
        // 注册一个Bean，指定bean名
        registry.registerBeanDefinition("person", beanDefinition);
      }
}
    ```



这是一个**非常重要**的注解，在Spring源码中，哪哪都能看到他的身影。

-   如 @EnableAspectJAutoProxy注解

    ```java
    @Target(ElementType.TYPE)
    @Retention(RetentionPolicy.RUNTIME)
    @Documented
    @Import(AspectJAutoProxyRegistrar.class)
    public @interface EnableAspectJAutoProxy {
        ...
    }
    ```

    EnableAspectJAutoProxy通过@Import引入了AspectJAutoProxyRegistrar类[实现了  ImportBeanDefinitionRegistrar]，这个Registrar里面又会向Spring容器中注册AnnotationAwareAspectJAutoProxyCreator（Spring aop注解实现的功臣）。

-   如 @EnableWebMvc注解。通过Import引入的是一个配置类

    ```java
    @Retention(RetentionPolicy.RUNTIME)
    @Target(ElementType.TYPE)
    @Documented
    @Import(DelegatingWebMvcConfiguration.class)
    public @interface EnableWebMvc {
    }
    ```

-   如 @EnableAsync注解。通过Import引入的是AsyncConfigurationSelector[实现了ImportSelector接口]

    ```java
    @Target(ElementType.TYPE)
    @Retention(RetentionPolicy.RUNTIME)
    @Documented
    @Import(AsyncConfigurationSelector.class)
    public @interface EnableAsync {
        ...
    }
    ```



