---
layout: post
title: Apollo源码阅读笔记（二）
categories: [Apollo]
description: Apollo源码学习 —— 配置自动更新
keywords: apollo, 携程apollo, ctripcorp apollo
---
## Apollo源码阅读笔记（二）

[前面](2018-12-20-apollo-source-code-reading-notes.md) 分析了apollo配置设置到Spring的environment的过程，此文继续PropertySourcesProcessor.postProcessBeanFactory里面调用的第二个方法initializeAutoUpdatePropertiesFeature(beanFactory)，其实也就是配置修改后更新相关处理逻辑。



在继续分析之前，先来看下配置是怎么自动更新的。
### 1. 配置更新简单示例

通过portal页面，修改配置之后，我们可以通过@ApolloConfigChangeListener来自己监听配置的变化手动更新，也可以通过@Value标记字段，字段值会自动更新。

#### 1.1 更新event相关对象

无论是哪种方式，都离不开event，所以先了解下event相关的对象

```java
// change事件
public class ConfigChangeEvent {
  private final String m_namespace;
  private final Map<String, ConfigChange> m_changes;
  // 省略了全部方法  
}

// change实体bean
public class ConfigChange {
  private final String namespace;
  private final String propertyName;
  private String oldValue;
  private String newValue;
  private PropertyChangeType changeType;
  // 省略了全部方法  
}

// change类型
public enum PropertyChangeType {
  ADDED, MODIFIED, DELETED
}
```

#### 1.2 Value注解方式

直接在属性字段上标记@Value就可以了。

eg：

```java
@Service
public class XXXService {
    @Value("${config.key}")
    private String XXConfig;
    ....
}
```

如果修改了config.key配置项，那么这里的XXConfig的值是会自动更新的。



#### 1.3 自定义ConfigChangeListener方式

@Value方式实现起来很简单，但如果要更灵活点，比如加上一些自己的业务处理，那就需要用到@ApolloConfigChangeListener了。

```java
@ApolloConfigChangeListener
private void onChange(ConfigChangeEvent changeEvent) {
	logger.info("配置参数发生变化[{}]", JSON.toJSONString(changeEvent));
	doSomething();
}
```

标记有@ApolloConfigChangeListener的这个方法，必须带一个ConfigChangeEvent的入参，通过这个event可以拿到事件的类型、变化的具体字段、变化前后值。

拿到event之后，我们可以根据具体的变化做不同业务处理。



以上是更新的使用，下面来深入研究下源码的实现。



### 2.  配置更新监听listener

先继续文章开头留下的尾巴 initializeAutoUpdatePropertiesFeature方法。

#### 2.1 AutoUpdateConfigChangeListener

PropertySourcesProcessor.postProcessBeanFactory –> initializeAutoUpdatePropertiesFeature中。具体逻辑如下：

1.  构造一个 AutoUpdateConfigChangeListener 对象 [implements ConfigChangeListener]；

2.  拿到前面处理的所有的ConfigPropertySource组成的list，遍历ConfigPropertySource，设置listener

    ​	configPropertySource.addChangeListener(autoUpdateConfigChangeListener);

    ​	这里加事件，最终是加在Config上了。



我们前面使用的@Value注解标记的字段，在字段值发生变化时，就是通过这里加的listener，收到通知的。

@Value是org.springframework.beans.factory.annotation.value，Spring的注解。apollo通过自己的SpringValueProcessor来处理它。来看下完整的流程：

1.  SpringValueProcessor继承了ApolloProcessor，间接实现了BeanPostProcessor

2.  在启动的时候，会被调到 postProcessBeforeInitialization，进而调到processField
    1.  判断字段是否被@Value标记，如果没有则返回，有则继续往下
    2.  解析@Value注解里面设置的value
    3.  构造SpringValue对象 new SpringValue(key, value.value(), bean, beanName, field, false)
    4.  将构造的springValue存到SpringValueRegistry的Map<BeanFactory, Multimap<String, SpringValue>>中

3.  当修改配置发布事件后，AutoUpdateConfigChangeListener就被触发onChange(ConfigChangeEvent changeEvent)事件

    1.  通过event拿到所有变更的keys

    2.  遍历keys，通过springValueRegistry.get(beanFactory, key) 拿到SpringValue集合对象

        ​	这里就是从前面的2.4的map里面获取的

    3.  判断配置是否真的发生了变化 shouldTriggerAutoUpdate(changeEvent, key)

    4.  遍历SpringValue集合，逐一通过反射改变字段的值

到这里，@Value的更新流程就清楚了。

下面来看看自定义listener是怎么通知更新的。



#### 2.2 ConfigChangeListener

更多的时候，我们是通过在方法上标记@ApolloConfigChangeListener来实现自己的监听处理。[例子见1.3代码]

通过@ApolloConfigChangeListener注解添加的监听方法，默认关注的application namespace下的全部配置项。

有关该注解的处理逻辑在 com.ctrip.framework.apollo.spring.annotation.ApolloAnnotationProcessor 中，我们重点关注如下代码段 processMethod ：

```java
protected void processMethod(final Object bean, String beanName, final Method method) {
  ApolloConfigChangeListener annotation = AnnotationUtils.findAnnotation(method, ApolloConfigChangeListener.class);
  if (annotation == null) {
    return;
  }
  Class<?>[] parameterTypes = method.getParameterTypes();
  Preconditions.checkArgument(parameterTypes.length == 1, "Invalid number of parameters: %s for method: %s, should be 1", parameterTypes.length, method);
  Preconditions.checkArgument(ConfigChangeEvent.class.isAssignableFrom(parameterTypes[0]), "Invalid parameter type: %s for method: %s, should be ConfigChangeEvent", parameterTypes[0], method);

  ReflectionUtils.makeAccessible(method);
  String[] namespaces = annotation.value();
  String[] annotatedInterestedKeys = annotation.interestedKeys();
  Set<String> interestedKeys = annotatedInterestedKeys.length > 0 ? Sets.newHashSet(annotatedInterestedKeys) : null;
  // 创建listener  
  ConfigChangeListener configChangeListener = new ConfigChangeListener() {
    @Override
    public void onChange(ConfigChangeEvent changeEvent) {
      ReflectionUtils.invokeMethod(method, bean, changeEvent);
    }
  };

  // 给config设置listener
  for (String namespace : namespaces) {
    Config config = ConfigService.getConfig(namespace);
    if (interestedKeys == null) {
      config.addChangeListener(configChangeListener);
    } else {
      config.addChangeListener(configChangeListener, interestedKeys);
    }
  }
}
```

经过这段代码处理，如果有change事件，我们通过@ApolloConfigChangeListener自定义的listener就会收到消息了。

前面了解完了监听，下面来看下事件的发布。



### 3.  配置更新事件发布

后台portal页面修改发布之后，client端怎么接收到事件呢？其实在client启动后，就会和服务端建一个长连接。代码见 com.ctrip.framework.apollo.internals.RemoteConfigRepository 。

先来看看RemoteConfigRepository 构造方法

```java
public RemoteConfigRepository(String namespace) {
    m_namespace = namespace;
    m_configCache = new AtomicReference<>();
    m_configUtil = ApolloInjector.getInstance(ConfigUtil.class);
    m_httpUtil = ApolloInjector.getInstance(HttpUtil.class);
    m_serviceLocator = ApolloInjector.getInstance(ConfigServiceLocator.class);
    remoteConfigLongPollService = ApolloInjector.getInstance(RemoteConfigLongPollService.class);
    m_longPollServiceDto = new AtomicReference<>();
    m_remoteMessages = new AtomicReference<>();
    m_loadConfigRateLimiter = RateLimiter.create(m_configUtil.getLoadConfigQPS());
    m_configNeedForceRefresh = new AtomicBoolean(true);
    m_loadConfigFailSchedulePolicy = new ExponentialSchedulePolicy(m_configUtil.getOnErrorRetryInterval(),
        m_configUtil.getOnErrorRetryInterval() * 8);
    gson = new Gson();
    this.trySync();
    this.schedulePeriodicRefresh();
    this.scheduleLongPollingRefresh();
  }
```

从上面构造方法最后几行可以看出，启动的时候，会先尝试从server端同步，然后会启动2个定时刷新任务，一个是定时刷新，一个是长轮询。不过不管是哪种方式，最终进入的还是 trySync() 方法。

以com.ctrip.framework.apollo.internals.RemoteConfigLongPollService为例

>   RemoteConfigLongPollService
>
>   —> startLongPolling() 
>
>   —> doLongPollingRefresh(appId, cluster, dataCenter)，发起http长连接  
>
>   —> 收到http response（server端发布了新的配置）
>
>   —> notify(lastServiceDto, response.getBody());
>
>   —> remoteConfigRepository.onLongPollNotified(lastServiceDto, remoteMessages);
>
>   —> 异步调用 remoteConfigRepository.trySync();

在trySync方法中，直接调用sync()后就返回，所以需要看sync()方法内部逻辑。在sync()方法内：

1.  先获取本机缓存的当前配置 ApolloConfig previous = m_configCache.get()

2.  获取server端的最新配置 ApolloConfig current = loadApolloConfig()

3.  如果 previous != current ，更新m_configCache

    ​	m_configCache.set(current);
    ​	this.fireRepositoryChange(m_namespace, this.getConfig());

4.  在fireRepositoryChange里面，遍历当前namespace下的listener，调用RepositoryChangeListener事件

    ​	listener.onRepositoryChange(namespace, newProperties) 

    ​		<—这里事件就传到**Local**FileConfigRepository	这一层了

5.  本机该namespace的LocalFileConfigRepository实现了RepositoryChangeListener，所以会受到通知调用

       在LocalFileConfigRepository的onRepositoryChange方法中：

    1.  比较newProperties.equals(m_fileProperties)，相同就直接return，否则继续往下

    2.  更新本机缓存文件 updateFileProperties

    3.  触发 fireRepositoryChange(namespace, newProperties); <—这里事件就传到**Config**这一层了

    4.  触发事件在 DefaultConfig 中得到响应处理。这里先对发生变化的配置做一些处理，然后发ConfigChangeEvent事件

        ​	this.fireConfigChange(new **ConfigChangeEvent**(m_namespace, actualChanges));

    5.  在DefaultConfig的this.fireConfigChange里面，就会遍历 listeners，依次调用onChange方法

        ​	准确的说，是在父类AbstractConfig中实现的；

        ​	这里的listeners就有前面提到的AutoUpdateConfigChangeListener 和 ApolloAnnotationProcessor 中定义的ConfigChangeListener

至此，事件的发布监听就形成闭环了，这里fireConfigChange(ConfigChangeEvent)后，@Value标记的字段、@ApolloConfigChangeListener都会被触发更新。




