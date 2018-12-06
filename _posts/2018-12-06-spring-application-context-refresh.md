#### Spring容器的创建刷新过程

以AnnotionConfigApplicationContext为例，在new一个AnnotionConfigApplicationContext的时候，其构造函数内就会调用父类的refresh方法

```java
public AnnotationConfigApplicationContext(Class<?>... annotatedClasses) {
    this();
    register(annotatedClasses);
    refresh();// <-- 调用AbstractApplicationContext的refresh方法
}
```

所以呢，Spring容器的创建过程主要在这个refresh方法里边。

```java
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        // 1、Prepare this context for refreshing.
        prepareRefresh();

        ///2、Tell the subclass to refresh the internal bean factory.
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        ///3、Prepare the bean factory for use in this context.
        prepareBeanFactory(beanFactory);

        try {
            //4、 Allows post-processing of the bean factory in context subclasses.
            postProcessBeanFactory(beanFactory);

            //5、 Invoke factory processors registered as beans in the context.
            invokeBeanFactoryPostProcessors(beanFactory);

            //6、 Register bean processors that intercept bean creation.
            registerBeanPostProcessors(beanFactory);

            //7、 Initialize message source for this context.
            initMessageSource();

            //8、 Initialize event multicaster for this context.
            initApplicationEventMulticaster();

            //9、 Initialize other special beans in specific context subclasses.
            onRefresh();

            //10、 Check for listener beans and register them.
            registerListeners();

            //11、 Instantiate all remaining (non-lazy-init) singletons.
            finishBeanFactoryInitialization(beanFactory);

            //12、 Last step: publish corresponding event.
            finishRefresh();
        } catch (BeansException ex) {
            if (logger.isWarnEnabled()) {
                logger.warn("Exception encountered during context initialization - cancelling refresh attempt: " + ex);
            }

            // Destroy already created singletons to avoid dangling resources.
            destroyBeans();

            // Reset 'active' flag.
            cancelRefresh(ex);

            // Propagate exception to caller.
            throw ex;
        } finally {
            // Reset common introspection caches in Spring's core, since we
            // might not ever need metadata for singleton beans anymore...
            resetCommonCaches();
        }
    }
}
```

我给源码的每一步都加了序号（1~12），下面来详细看看每一步都做了些什么。

##### 1、prepareRefresh() 刷新前准备

1.  设置一些状态标记，如启动时间startupDate，closed/active状态

2.  initPropertySources() 设置init属性，此方法交给子类去实现

3.  getEnvironment().validateRequiredProperties() 

    如果无Environment，则new StandardEnvironment()

    再调用validateRequiredProperties() 校验参数

4.  earlyApplicationEvents= new LinkedHashSet<ApplicationEvent>();保存容器中的一些早期的事件


##### 2、obtainFreshBeanFactory() 获取BeanFactory

1.  refreshBeanFactory() 刷新BeanFactory

    在**GenericApplicationContext**的构造方法中new了一个DefaultListableBeanFactory

    通过this.beanFactory.setSerializationId(getId());设置一个唯一的id

2.  getBeanFactory() 返回刚才GenericApplicationContext创建的BeanFactory对象


##### 3、prepareBeanFactory(beanFactory) BeanFactory的准备工作

1.  设置BeanFactory的类加载器、表达式解析器【StandardBeanExpressionResolver】、ResourceEditorRegistrar

2.  添加BeanPostProcessor【ApplicationContextAwareProcessor】

3.  设置不需要自动装配的接口

    包括【EnvironmentAware、EmbeddedValueResolverAware、ResourceLoaderAware、ApplicationEventPublisherAware、MessageSourceAware、ApplicationContextAware】

4.  注册可以解析的自动装配的接口

    包括【BeanFactory、ResourceLoader、ApplicationEventPublisher、ApplicationContext】

5.  添加BeanPostProcessor【ApplicationListenerDetector】

6.  添加AspectJ【类加载时织入LoadTimeWeaver】

7.  给BeanFactory中注册一些能用的组件

    1.  environment【ConfigurableEnvironment】

    2.  systemProperties【Map<String, Object>】

    3.  systemEnvironment【Map<String, Object>】

##### 4、postProcessBeanFactory(beanFactory) BeanFactory后置处理

  子类通过重写这个方法来在BeanFactory创建并预准备完成**以后**做进一步的设置



<u>到这里，BeanFactory的创建及后置处理工作就结束了</u>



##### 5、invokeBeanFactoryPostProcessors(beanFactory) 执行BeanFactory后置方法

BeanFactoryPostProcessors是BeanFactory的后置处理器，在BeanFactory标准初始化之后、全部bean信息都被加载，但是还没有被实例化的时候执行。

在**invokeBeanFactoryPostProcessors**方法中，主要处理2种类型的接口：

>  BeanDefinitionRegistryPostProcessor 和 BeanFactoryPostProcessor

■先执行BeanDefinitionRegistryPostProcessor的方法，过程如下：

1.  获取所有实现了BeanDefinitionRegistryPostProcessor接口的class

    ```java
    beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false)
    ```

2.  先执行实现了PriorityOrdered优先级接口的BeanDefinitionRegistryPostProcessor

    ```java
    postProcessor.postProcessBeanDefinitionRegistry(registry)
    ```

3.  同理，再执行实现了Order顺序接口的BeanDefinitionRegistryPostProcessor；最后执行其他实现了BeanDefinitionRegistryPostProcessor的

> Configuration类中通过@Import(ImportBeanDefinitionRegistrar)引入的类就是在这里被调用registerBeanDefinitions方法的…【processor：ConfigurationClassPostProcessor】
  ```java
public void registerBeanDefinitions(
      AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry)
  ```



■再执行BeanFactoryPostProcessor的方法，过程如下：
1.  获取所有实现了BeanFactoryPostProcessor接口的class

    ```java
    beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false)
    ```

2.  先执行实现了PriorityOrdered优先级接口的BeanFactoryPostProcessor

    ```java
    postProcessor.postProcessBeanFactory(beanFactory)
    ```

3.  同理，再执行实现了Order顺序接口的BeanFactoryPostProcessor；最后执行其他实现了BeanDefinitionRegistryPostProcessor的


##### 6、registerBeanPostProcessors(beanFactory) 注册bean的后置处理器

1.  获取所有实现了BeanPostProcessor接口的class

    ```java
    beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);
    ```

2.  把获取的类按照 是否实现了 PriorityOrdered、Ordered接口、及其他 分成3类

3.  按照优先级依次调用

    ```java
    beanFactory.addBeanPostProcessor(postProcessor);
    ```

4.  最后注册一个ApplicationListenerDetector

    ```java
    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));
    ```

这个Detector的作用是：
> 在Bean创建完成后检查是否是ApplicationListener，如果是则加到context中applicationContext.addApplicationListener((ApplicationListener<?>) bean)



##### 7、initMessageSource() 初始化MessageSource组件

MessageSource组件主要用来实现国际化、消息解析处理。

在initMessageSource中，首先看容器中是否有id为messageSource的，类型是MessageSource的组件。如果没有则new DelegatingMessageSource() 放进去。

后面在处理国际化时，可以注入MessageSource对象，然后使用如下代码进行国际化

```
String getMessage(String code, Object[] args, Locale locale)
```



##### 8、initApplicationEventMulticaster() 初始化事件多波器

在此方法中，首选从BeanFactory中获取id为“applicationEventMulticaster”的ApplicationEventMulticaster。如果没有就new SimpleApplicationEventMulticaster(beanFactory)放进去。[多波器，有的也叫派发器]



##### 9、onRefresh() 留给子类重写



##### 10、registerListeners() 注册事件监听器

1.  从容器中拿到所有的ApplicationListener

2.  将每个监听器添加到事件多波器中

    ```
    getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName)
    ```

3.  处理前面步骤留下的earlyApplicationEvents，earlyApplicationEvents在步骤1.4中初始化为空的LinkedHashSet

    ```
    getApplicationEventMulticaster().multicastEvent(earlyEvent)
    ```



##### 11、finishBeanFactoryInitialization(beanFactory) 初始化所有剩下的单实例bean

核心逻辑在beanFactory.preInstantiateSingletons()方法中。这里面做的事情主要有：

1.  循环编译所有的beanNames，获取RootBeanDefinition

2.  如果bean 不是抽象的，是单实例的，是懒加载

3.  判断是否 是实现FactoryBean接口的Bean

    如果是，先做一些处理，再调getBean

    如果不是，则直接调用getBean(beanName) 

4.  getBean(beanName) –> doGetBean(name, null, null, false)

    1.  先获取缓存中保存的单实例Bean【singletonObjects.get(beanName)】。如果能获取到说明这个Bean之前被创建过（所有创建过的单实例Bean都会被缓存在ConcurrentHashMap<String, Object>(256)中）

    2.  如果缓存中获取不到，则开启下面的创建流程

    3.  先将bean标记为已创建 markBeanAsCreated(beanName)

    4.  获取Bean的定义信息RootBeanDefinition

    5.  获取当前Bean依赖的其他Bean[mbd.getDependsOn()]，如果有则调用getBean()把依赖的Bean先创建出来

    6.  开启单实例Bean的创建流程 createBean(beanName, mbd, args)

        1.  准备重写的方法

        2.  尝试返回代理对象

            ```java
            // Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
            Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
            ```

        3.  执行InstantiationAwareBeanPostProcessor

            先触发：postProcessBeforeInstantiation()；
            如果BeforeInstantiation有返回值，再接着执行postProcessAfterInitialization()；

            如果resolveBeforeInstantiation返回的不为null，则bean就创建好了

        4.  前面resolveBeforeInstantiation返回的不为null，则返回该bean；为null接着调Object beanInstance = doCreateBean(beanName, mbdToUse, args);创建Bean【step5】

5.  创建doCreateBean(beanName, mbdToUse, args)

    1.  创建instanceWrapper = createBeanInstance(beanName, mbd, args)

        利用工厂方法 或 对象的构造器 创建出Bean实例

    2.  调用MergedBeanDefinitionPostProcessor的postProcessMergedBeanDefinition(mbd, beanType, beanName);

    3.  给bean属性赋值 populateBean(beanName, mbd, instanceWrapper)

        1.  赋值前，拿到所有的InstantiationAwareBeanPostProcessor后置处理器，调用postProcessAfterInstantiation()

        2.  再拿到所有的InstantiationAwareBeanPostProcessor后置处理器，调用postProcessPropertyValues()

        3.  赋值 applyPropertyValues(beanName, mbd, bw, pvs);

            为属性利用setter方法等进行赋值



6.  初始化bean，调用initializeBean(beanName, exposedObject, mbd)

    1.  执行xxxAware方法 invokeAwareMethods(beanName, bean)，包括【BeanNameAware\BeanClassLoaderAware\BeanFactoryAware】

    2.  执行Bean后置处理器的before方法  applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName) 循环调用 beanProcessor.postProcessBeforeInitialization(result, beanName)

    3.  执行bean的初始化方法 invokeInitMethods(beanName, wrappedBean, mbd);

    4.  执行bean的后置处理器的after方法 applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName)  循环调用 beanProcessor.postProcessAfterInitialization(result, beanName)

        > Spring的AOP注解实现原理：
        >
        > @EnableAspectJAutoProxy引入了继承自AbstractAutoProxyCreator的AnnotationAwareAspectJAutoProxyCreator
        >
        > 在AbstractAutoProxyCreator中实现了postProcessAfterInitialization方法，在方法内调用wrapIfNecessary(bean, beanName, cacheKey)方法创建代理对象返回出去，后续放入容器的就是这个代理对象

    5.  注册bean的销毁方法 registerDisposableBeanIfNecessary(beanName, bean, mbd);



7.  将bean加入到singletonObjects中

8.  所有bean创建完之后，判断bean是否是SmartInitializingSingleton的实例。如果是，就执行afterSingletonsInstantiated()


##### 12、finishRefresh() 容器创建完成

1.  initLifecycleProcessor() 初始化和生命周期有关的后置处理器

    默认从容器中找是否有lifecycleProcessor的组件【LifecycleProcessor】；如果没有new DefaultLifecycleProcessor();加入到容器；

2.  执行onfresh方法

    ```java
    getLifecycleProcessor().onRefresh()
    ```

3.  发布容器刷新完成事件

    ```
    publishEvent(new ContextRefreshedEvent(this))
    ```

4.  调用LiveBeansView.registerApplicationContext(this)

