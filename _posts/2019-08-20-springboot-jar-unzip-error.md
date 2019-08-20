---
layout: post
title: springboot打成jar包后无法解压
categories: [Springboot]
description: springboot打成jar包后无法解压
keywords: Springboot, jar, 解压
---

### springboot打成jar包后无法解压

Springboot打出来的jar，用压缩工具解压报错。Why? 先说解决办法。

#### 1、解决办法

>   executable属性导致的，属性改成false后重新打包，就可以解压

```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <executable>true</executable>
    </configuration>
</plugin>
```

那么，executable设置成true作用是什么呢？为什么设置成true就无法解压呢？



#### 2、executable设置成true作用

一般情况下，我们运行jar的方式为：

>   java -jar xxx.jar

但如果你想在unix/linux上，像执行某个sh/服务那样运行jar，就需要把你的app打包成executable的jar。

官方解释：

>   In addition to running Spring Boot applications by using `java -jar`, it is also possible to make fully executable applications for Unix systems. A fully executable jar can be executed like any other executable binary or it can be [registered with `init.d` or `systemd`](https://docs.spring.io/spring-boot/docs/current/reference/html/deployment-install.html#deployment-service). This makes it very easy to install and manage Spring Boot applications in common production environments.

【A fully executable jar】就是通过前面提到的，打包时把executable设置为true。



#### 3、无法解压的原因分析

完全可执行 的 jar/war 在文件前面嵌入了个 额外的脚本，这就使得有些命令会执行失败，比如 jar -xf 等。

这就是为什么解压工具无法解压的原因。

所以，如果你的jar是通过【java -jar】执行、或 放在servlet容器中执行，那么建议将executable设置为false。

>   Fully executable jars work by embedding an extra script at the front of the file. Currently, some tools do not accept this format, so you may not always be able to use this technique. For example, `jar -xf` may silently fail to extract a jar or war that has been made fully executable. It is recommended that you make your jar or war fully executable only if you intend to execute it directly, rather than running it with `java -jar` or deploying it to a servlet container.



#### 4、参考资料

https://docs.spring.io/spring-boot/docs/current/reference/html/deployment-install.html#deployment-service





