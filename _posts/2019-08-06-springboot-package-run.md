---
layout: post
title: Springboot打包执行源码解析
categories: [Springboot]
description: Springboot打包执行源码解析
keywords: springboot,打包,执行,spring-boot-maven-plugin
---
### 一、打包
Springboot打包的时候，需要配置一个maven插件[spring-boot-maven-plugin]

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>
```
这个插件提供了5个功能模块，包括：
- build-info：生成项目的构建信息文件build-info.properties
- repackage：默认goal。在mvn package执行之后，再次打包生成可执行的jar/war，同时重命名mvn package生成的jar/war为 ***.origin
- run：这个可以用来运行Spring Boot应用
- start：这个在 mvn integration-test 阶段，进行 Spring Boot 应用生命周期的管理
- stop：这个在 mvn integration-test 阶段，进行 Spring Boot 应用生命周期的管理

如果想mvn package打的包不被重命名，可以配置classifier，这样Springboot打包生成的可执行jar就是XXX-executable.jar了。
```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <classifier>executable</classifier>
                </configuration>
        </plugin>
    </plugins>
</build>
```

### 二、区别
Springboot打的包和Maven打的包区别在哪里呢？  
把maven package打的包 a.jar.original 重命名成a-original.jar；然后和Springboot打的包a.jar比较一下，发现：  
Springboot打的包的MANIFEST.MF文件多了如下几行：

```
...
Main-Class: org.springframework.boot.loader.JarLauncher
Start-Class: com.xxx.XxxApplication
...
```
这里的Start-Class就是我们自己写的代码的main入口类了；而Main-Class是Springboot给我们添加的启动类；  


###  三、源码Debug
要了解Springboot可执行jar的执行过程，最好的途径就是debug一下。下面就先配置一下。  

1. 设置执行jar的命令
    > 一般执行jar使用的命令是 java -jar xxx.jar   
    > debug需要开调试端口，命令是 java -agentlib:jdwp=transport=dt_socket,server=y,address=5005,suspend=y -jar xxx.jar

2. 在idea中配置remote debug，设置端口号为5005，Host为localhost
3. 在项目的pom.xml中，增加spring-boot-loader的maven依赖
    ```
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-loader</artifactId>
        <version>1.5.10.RELEASE</version>
    </dependency>
    ```
4. 完成上述配置之后，先debug启动程序，在打开idea的调试


### 四、源码解析

1. 先看看前面提说到的Main-Class
   即org.springframework.boot.loader.JarLauncher
    ```java
    public class JarLauncher extends ExecutableArchiveLauncher {

    	static final String BOOT_INF_CLASSES = "BOOT-INF/classes/";
    
    	static final String BOOT_INF_LIB = "BOOT-INF/lib/";
    
    	public JarLauncher() {
    	}
    
    	protected JarLauncher(Archive archive) {
    		super(archive);
    	}
    
    	@Override
    	protected boolean isNestedArchive(Archive.Entry entry) {
    		if (entry.isDirectory()) {
    			return entry.getName().equals(BOOT_INF_CLASSES);
    		}
    		return entry.getName().startsWith(BOOT_INF_LIB);
    	}
    
    	public static void main(String[] args) throws Exception {
    		new JarLauncher().launch(args);
    	}

    }
    ```

2. 跟着进父抽象类Launcher的launch方法

    在这个方法里面，通过getClassPathArchives()把jar包里的jar抽象成Archive对象列表。 
    在Springboot loader中，抽象出了Archive概念。 
    一个archive可以是一个jar（JarFileArchive），也可以是一个文件目录（ExplodedArchive）。这些都可以理解为Springboot抽象出来的统一访问资源的层。
    此List<Archive>包括：

    - \BOOT-INF\classes （项目的class）  
    - \BOOT-INF\lib 目录下的所有jar  


3. 遍历List<Archive>，获得每个Archive的URL，组成List<URL>
4. 创建一个自定义的类加载器LaunchedURLClassLoader。这个类加载器继承自jdk自带的java.net.URLClassLoader
5. 加载，创建MainMethodRunner
    看下Launcher的方法，逻辑很清晰
    ```java
    public abstract class Launcher {
    
        /**
         * 1、获取List<Archive>
         * 2、创建LaunchedURLClassLoader
         * 3、找到main class
         * 4、加载
         */
        protected void launch(String[] args) throws Exception {
            JarFile.registerUrlProtocolHandler();
            ClassLoader classLoader = createClassLoader(getClassPathArchives());
            launch(args, getMainClass(), classLoader);
        }
    
        /**
         * 创建classloader
         */
        protected ClassLoader createClassLoader(List<Archive> archives) throws Exception {
            List<URL> urls = new ArrayList<URL>(archives.size());
            for (Archive archive : archives) {
                urls.add(archive.getUrl());
            }
            return createClassLoader(urls.toArray(new URL[urls.size()]));
        }
    
        /**
         * 创建LaunchedURLClassLoader
         */
        protected ClassLoader createClassLoader(URL[] urls) throws Exception {
            return new LaunchedURLClassLoader(urls, getClass().getClassLoader());
        }
    
        /**
         * 通过指定的classloader，加载main class
         */
        protected void launch(String[] args, String mainClass, ClassLoader classLoader)
                throws Exception {
            Thread.currentThread().setContextClassLoader(classLoader);
            createMainMethodRunner(mainClass, args, classLoader).run();
        }
    
        /**
         * 创建MainMethodRunner
         */
        protected MainMethodRunner createMainMethodRunner(String mainClass, String[] args, ClassLoader classLoader) {
            return new MainMethodRunner(mainClass, args);
        }
    
        /**
         * 抽象方法，用来获取main class
         */
        protected abstract String getMainClass() throws Exception;
    
        /**
         * 抽象方法，用来获取List<Archive>
         */
        protected abstract List<Archive> getClassPathArchives() throws Exception;
    
        protected final Archive createArchive() throws Exception {
            ProtectionDomain protectionDomain = getClass().getProtectionDomain();
            CodeSource codeSource = protectionDomain.getCodeSource();
            URI location = (codeSource == null ? null : codeSource.getLocation().toURI());
            String path = (location == null ? null : location.getSchemeSpecificPart());
            if (path == null) {
                throw new IllegalStateException("Unable to determine code source archive");
            }
            File root = new File(path);
            if (!root.exists()) {
                throw new IllegalStateException(
                        "Unable to determine code source archive from " + root);
            }
            return (root.isDirectory() ? new ExplodedArchive(root)
                    : new JarFileArchive(root));
        }
}
    ```
6. 执行MainMethodRunner的run方法，启动主类的main

    ```java
    public class MainMethodRunner {
    
        private final String mainClassName;
    
        private final String[] args;
    
        public MainMethodRunner(String mainClass, String[] args) {
            this.mainClassName = mainClass;
            this.args = (args == null ? null : args.clone());
        }
    
        /**
         * 通过反射找到main方法，然后invoke
         */
        public void run() throws Exception {
            Class<?> mainClass = Thread.currentThread().getContextClassLoader()
                    .loadClass(this.mainClassName);
            Method mainMethod = mainClass.getDeclaredMethod("main", String[].class);
            mainMethod.invoke(null, new Object[] { this.args });
        }
}

    ```

