---
layout: post
title: SpringCloud注册中心搭建使用
categories: [SpringCloud]
description: SpringCloud之注册中心
keywords: SpringCloud, eureka, Registry Center
---

## 一、SpringCloud注册中心搭建

注册中心使用Eureka<br />
早期artifactId使用spring-cloud-starter-eureka-server<br />但是新版需要使用spring-cloud-starter-netflix-eureka-server。

#### 1. 新建一个maven工程
#### 2. pom文件引入如下依赖
```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<groupId>com.ye</groupId>
	<artifactId>eureka-server</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<name>eureka-server</name>
	<description>eureka-server project for Spring Boot</description>

	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>2.0.3.RELEASE</version>
		<relativePath />
	</parent>

	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
		<java.version>1.8</java.version>
		<spring-cloud.version>Finchley.RELEASE</spring-cloud.version>
	</properties>

	<dependencies>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
	</dependencies>

	<dependencyManagement>
		<dependencies>
			<dependency>
				<groupId>org.springframework.cloud</groupId>
				<artifactId>spring-cloud-dependencies</artifactId>
				<version>${spring-cloud.version}</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
		</dependencies>
	</dependencyManagement>

	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>

	<repositories>
		<repository>
			<id>spring-milestones</id>
			<name>Spring Milestones</name>
			<url>https://repo.spring.io/milestone</url>
			<snapshots>
				<enabled>false</enabled>
			</snapshots>
		</repository>
	</repositories>

</project>
```
#### 3. 配置文件application.yml
```yml
server:
  port: 8888

eureka:
  instance:
    hostname: localhost
  client:
    #是否向注册中心注册自己【自己就是注册中心，所以不需要注册自己】
    registerWithEureka: false
    #是否获取eureka注册中心上的注册信息【自己的职责就是维护服务，并不需要去检索服务】       
    fetchRegistry: false            
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
```

#### 4. 启动类
```
@EnableEurekaServer
@SpringBootApplication
public class EurekaServerApp {

	public static void main(String[] args) {
		SpringApplication.run(EurekaServerApp.class, args);
	}
}
```

#### 5. 启动

  运行EurekaServerApp，在浏览器输入http://localhost:8888/ 即打开到配置中心页面。<br />
此时，因为没有服务注册过来，所以列表是空的，后文再来创建一个服务注册到Eureka中来。

#### 6. 注册中心高可用

  注册中心高可用的时候，就需要将注册中心相互注册，此时就需要设置registerWithEureka和fetchRegistry都为true。配置文件如下：
```
server:
  port: 9998

spring:
  application:
    name: eureka-group-server

eureka:
  instance:
    hostname: peer1
  client:
    registerWithEureka: true       
    fetchRegistry: true            
    serviceUrl:
      defaultZone: http://peer3:9997/eureka/,http://peer2:9999/eureka/
```
peer1/peer2/peer3分别指向3个注册中心的地址，他们之间需相互注册。


## 二、服务搭建
#### 1. 新建一个maven project
#### 2. pom文件中引入如下依赖
```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
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
#### 3. 配置文件

```
server:
  port: 8889 
  
spring:
  application:
    name: demo-service

eureka:
  instance:
    ip-address: 127.0.0.1
    prefer-ip-address: true
    # 指定注册到注册中心的id，注册中心列表会展示成  127.0.0.1：8889：demo-service
    instance-id: ${eureka.instance.ip-address}:${server.port}:${spring.application.name}
    
  client:
    serviceUrl:
      # 注册中心地址
      #defaultZone: http://peer1:9998/eureka/,http://peer2:9999/eureka/,http://peer3:9997/eureka/
      defaultZone: http://localhost:8888/eureka/
```
#### 4. 启动类

```
@EnableEurekaClient
@SpringBootApplication
public class DemoServiceApp {

	public static void main(String[] args) {
		SpringApplication.run(DemoServiceApp.class, args);
	}
}
```
#### 5. 启动
运行DemoServiceApp之后，注册中心就能看到demoservice的服务了


## 三、 其他说明
spring cloud中discovery service有许多种实现（eureka、consul、zookeeper等等）<br/>@EnableDiscoveryClient基于spring-cloud-commons<br/>@EnableEurekaClient基于spring-cloud-netflix。<br/>
如果选用的注册中心是eureka，那么就推荐@EnableEurekaClient，如果是其他的注册中心，那么推荐使用@EnableDiscoveryClient。<br/>
参考：<https://blog.csdn.net/hh652400660/article/details/79474419>

　
　　
　　　



