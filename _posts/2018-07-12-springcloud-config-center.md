---
layout: post
title: SpringCloud本地配置中心
categories: [SpringCloud]
description: SpringCloud之配置中心
keywords: SpringCloud, eureka, Config Center, SpringBoot
---

## 一、SpringCloud本地配置中心搭建
spring cloud配置中心可以将配置放在git、svn行，本文主要是把配置放在本机上。

#### 1、新建一个maven 工程
#### 2、引入如下依赖
```
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-config-server</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```
从上面也能看出，配置中心也将自己注册到注册中心上

#### 3、配置文件
```
server:
  port: 1111

spring:
  application:
    name: config-server
  profiles:
    active: native   # 开启本地配置
    
eureka:
  instance:
    ip-address: 127.0.0.1
    instance-id: ${eureka.instance.ip-address}:${server.port}:${spring.application.name}
    prefer-ip-address: true
    
  client:
    serviceUrl:
      defaultZone: http://localhost:8888/eureka/    
```
在未设置spring.cloud.config.server.native.search-locations时，默认读取的配置文件目录为［src/main/resources］。<br/>


#### 4、启动类
```
@EnableDiscoveryClient
@EnableConfigServer
@SpringBootApplication
public class ConfigServerApplication {
	public static void main(String[] args) {
		SpringApplication.run(ConfigServerApplication.class, args);
	}
}
```



## 二、使用配置中心
#### 1、新建一个maven工程

#### 2、引入如下依赖
```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-config</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```
上面引入了actuator，他提供了refresh接口，可以用来刷新配置文件

#### 3、配置文件
注意：配置文件名必须是bootstrap.yml或者bootstrap.properties
```
server:
  port: 7002
spring:
  application:
    name: config-client
  cloud:
    config:
      discovery:
        enabled: true
        service-id: config-server
      failFast: true
      profile: dev

eureka:
  instance:
    ip-address: 127.0.0.1
    prefer-ip-address: true
    instance-id: ${eureka.instance.ip-address}:${server.port}:${spring.application.name}
  client:
    serviceUrl:
      defaultZone: http://localhost:8888/eureka/
      
      
management:
  endpoints:
    web:
      exposure:
        include: refresh,health,info
```



#### 4、获取配置
可以直接使用@Value("${test.key1}")来获取，也可以创建一个配置对象，然后通过配置对象提供的get方法来获取。代码如下：
```
@RefreshScope
@Component
@ConfigurationProperties(prefix = "test")
public class ConfigProp {
	private String key1;
	private String key2;
	// 省略了get、set方法
}
```


#### 5、启动类
```
@EnableDiscoveryClient
@SpringBootApplication
public class ConfigClientApp {
	
	public static void main(String[] args) {
		SpringApplication.run(ConfigClientApp.class, args);
	}
}
```


## 三、其他
#### 1、配置中心指定本地配置文件目录
```
spring:
  cloud:
    config:
      server:
        native:
          search-locations:
          - F:/spring-tool-suite-3.9.4.RELEASE-e4.7.3a-win32-x86_64/config_center_file
          - file:/home/dz-m/dz-m-config  # linux环境
```
需要注意的是：
- 默认目录为［src/main/resources］,但是如果你指定到这个目录，例如[F:\workspace_sts\config-server\src\main\resources]，它会将配置中心的配置文件[application.yml]也当做是配置。此时启动config-client的时候，会提示端口被占用。因为client端把server端的端口读取过来当做配置项使用了。
#### 2、config-client配置文件名
config-client中使用的配置文件，文件名==一定要是bootstrap.yml或bootstrap.properties==。<br/>因为bootstrap.properties的加载是先于application.properties的。 [原因参考](https://blog.csdn.net/u012076316/article/details/53323138)
<br />
config-server中配置文件可以用application.yml 或 application.properties


