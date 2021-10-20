---
layout: post
title: springboot应用多环境配置-特殊场景
categories: [Springboot]
description: springboot自定义配置文件多环境配置-特殊场景
keywords: springboot, maven, profile
---

# springboot应用多环境配置



## 1、常见场景

Springboot本身支持多环境配置，而且相当简单，直接在resources文件夹下创建三个以properties为后缀的文件就可以了

-    application-dev.properties：开发环境
-    application-test.properties：测试环境
-    application-prod.properties：生产环境

我今天要说的是一个稍特殊的场景。



## 2、特殊场景

公司有个统一的SSO登录服务器，对各个应用提供了sso-client.jar，而且在使用的时候，应用里面需要带一个名为“sso.properties”的配置里面，里面配置sso相关信息。大致这样子滴~

```properties
#接入SSO认证中心的应用ID
clientAppId=
#SSO认证中心分配给本应用的签名密钥
signKey=
#SSO认证中心网站域名
serverHost=
```

而读取这个配置文件的代码长这样：

```java
public class SSOPropertiesUtils {
    private static final Logger logger = Logger.getLogger(SSOPropertiesUtils.class);
    public static final String propertiesFileName = "sso.properties";
    private static Properties ssoProperties = new Properties();

    public SSOPropertiesUtils() {
    }
    public static String getPropertie(String key) {
        return ssoProperties.getProperty(key);
    }

    static {
        InputStream resourceAsStream = SSOPropertiesUtils.class.getClassLoader().getResourceAsStream("sso.properties");

        try {
            ssoProperties.load(resourceAsStream);
        } catch (IOException var3) {
            String message = "统一认证配置文件sso.properties不存在或错误";
            logger.error(message, var3);
            throw new RuntimeException(message);
        }
    }
}
```

额，这里代码写死了，我只能在resources根目录放一个 sso.properties 配置文件了。

但是，测试环境和生产环境的应用id是不一样的呀~ ~ 怎么让他读不同的文件呢？



### 2.1、方法一 ：mvn打包时设置不同的值

首先，在resources根目录，新建一个sso.properties文件，里面的值设置成占位符的形式

```properties
clientAppId=@clientAppId@
signKey=@signKey@
serverHost=@serverHost@
```

然后，在pom文件中，定义profiles。注意&lt;clientAppId&gt;和@clientAppId@里面的key要对应

```xml
<profiles>
    <profile>
        <id>dev</id>
        <properties>
            <clientAppId>aaa</clientAppId>
            <signKey>bbb</signKey>
            <serverHost>http://www.baidu.com</serverHost>
        </properties>
        <activation>
            <activeByDefault>true</activeByDefault> <!-- 设置默认profile -->
        </activation>
    </profile>

    <profile>
        <id>prod</id>
        <properties>
            <clientAppId>aaa2</clientAppId>
            <signKey>bbb2</signKey>
            <serverHost>http://www.baidu2.com</serverHost>
        </properties>
    </profile>
</profiles>


<build>
    <resources>
        <resource>
            <directory>${basedir}/src/main/resources</directory>
            <filtering>true</filtering> <!-- 这行必须 -->
            <includes>
                <include>**/**</include>
            </includes>
        </resource>
    </resources>
</build>
```

最后，在maven打包的时候，带上 -P dev 或 -P prod 指定对应的profile生效

```
mvn clean package -DskipTests -Pdev -X
```

如果没有生效，检查2点：

1.  设置filtering为true

2.  检查配置文件中写的占位符用的分隔符。我上面写成 @clientAppId@ 是因为spring-boot-starter-parent.pom文件中，定义的是 

    >   &lt;resource.delimiter&gt;@&lt;/resource.delimiter&gt;



### 2.2、方法二：通过反射修改这里的值

从前面的代码里面可以看到，SSOPropertiesUtils是从ssoProperties属性中取值的，我们只要改了这个ssoProperties就可以改变值了。那么可以这么做：

1.  定义一个PropertiesConfiguration，大致如下：

```java
@Configuration
public class PropertiesConfiguration implements ApplicationContextInitializer<ConfigurableApplicationContext> {
    public PropertiesConfiguration() {
    }

    @Override
    public void initialize(ConfigurableApplicationContext context) {
        try {
            Config config = ConfigService.getConfig("sso");
            Properties properties = new Properties();
            Set<String> propertyNames = config.getPropertyNames();
            propertyNames.stream().forEach(x -> {
                String property = config.getProperty(x, "");
                properties.put(x, property);
            });
			// 上面是从apollo配置中心取配置，你也可以自定义
            Field ssoProperties = ReflectionUtils.getDeclaredField(SSOPropertiesUtils.class, "ssoProperties");
            ssoProperties.setAccessible(true);
            ssoProperties.set(null, properties);
        } catch (Throwable e) {
            throw new RuntimeException("无法设置sso.properties", e);
        }
    }
}
```

2. 在 resources/META-INF 下新增一个 spring.factories，内容如下：

```properties
org.springframework.context.ApplicationContextInitializer=\
com.yejg.PropertiesConfiguration
```

3. 为了保证启动不报错，还要在 resource 下面新建一个sso.properties，里面是空的

通过这种方式也可以达到目的。
