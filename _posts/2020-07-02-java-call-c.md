---
layout: post
title: java调用so
categories: [Java]
description: java调用so的方法
keywords: jni,jna
---
昨天接到个小需求，需要在java中调第三方的so。回想上一次使用jni还是刚毕业那会儿，那时候我还会自己写C，生成dll和so，然后通过jni来调。惭愧，现在C/C++已经完全不会了…

使用原生的jni开发略麻烦，可以直接基于jna(java native access)这个jar。具体步骤如下：

### 引入jna的jar

```
<dependency>
    <groupId>com.sun.jna</groupId>
    <artifactId>jna</artifactId>
    <version>3.0.9</version>
</dependency>
```

 

### 定义一个接口和so中的方法对应

比如我这边拿到的.h文件为

>   int decryptToken(const char *sIn, int iInlen);

那么我的java代码就可以这么写

```java
import com.sun.jna.Library;
import com.sun.jna.Native;

public interface DecryptLib extends Library {
    DecryptLib INSTANCE = (DecryptLib) Native.loadLibrary("/usr/local/libtoken.so", DecryptLib.class);
    int decryptToken(String encrptyToken, int length);
}
```

本想把so放在jar里面，省的部署环境的时候容易漏掉。但一直没找到简单的方法，网上说了一堆，但是我验证了不行。一般都还是需要先解压到某个临时目录。

See <https://stackoverflow.com/questions/4113317/load-library-from-jar>



### 使用

```java
public class DecryptUtil {
    private static DecryptLib instance = DecryptLib.INSTANCE;

    public static void decrypt(String token) {
        int code = instance.decryptToken(token, token.length());
    }
}
```



### Java和C数据类型对应关系

<这个网上太多了，不抄过来了，^_^>



### 可能遇到的问题

（1）提示找不到

>   java.lang.UnsatisfiedLinkError: Unable to load library 'libtoken': libtoken.so: cannot open shared object file: No such file or directory

这种情况一般都是JVM找不到动态链接库导致的，检查下路径。

上面的代码是写的so的绝对路径，如果你写的相对路径，并添加了classpath，那么这里只需要写

```
DecryptLib INSTANCE = (DecryptLib) Native.loadLibrary("token", DecryptLib.class);
```

不需要lib前缀，也不需要扩展名.so



（2）提示找不到方法

>   java.lang.UnsatisfiedLinkError: Error looking up function 'decryptToken': /usr/loca/libtoken.so: undefined symbol: decryptToken

dll/so写法问题。原因是，C++中，方法必须加上extern “C”，否则无法找到c++方法。



（3）提示没有GLIBCXX_3.4.18

>   java.lang.UnsatisfiedLinkError: Unable to load library '/usr/local/libtoken.so': /usr/lib64/libstdc++.so.6: version `GLIBCXX_3.4.18' not found (required by /usr/local/libtoken.so)

这种一般是so需要依赖对应版本的glibcxx，安装一下就好了。

可以通过【strings /usr/lib64/libstdc++.so.6 | grep GLIBC】命令看下当前linux机器上已有的版本

