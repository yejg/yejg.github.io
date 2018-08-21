---
layout: post
title: Eclipse插件改造——AlibabaSmartfoxEclipse插件源码编译
categories: [eclipse plug-in, p3c]
description: Alibaba smartfox-eclipse插件源码编译
keywords: eclipse, plugin, plug-in, p3c, smartfox
---
### smartfox-eclipse插件源码编译
Alibaba的代码检查插件有2个版本：eclipse和idea，此文是eclipse版本的插件编译说明。

#### 项目开源地址
https://github.com/alibaba/p3c

插件安装地址：
https://p3c.alibaba.com/plugin/eclipse/update

#### 环境准备
- maven 3+
- Tycho环境准备
  > 配置Tycho Configurator，参考[用Tycho构建RPC程序](http://chnic.iteye.com/blog/2201139)
- 安装Kotlin插件
  > 在eclipse market中搜索Kotlin，install，重启


#### 导入源码
按照maven project的方式导入即可。   
导入之后，应该会报错
![image](/images/posts/eclipse-plugin/smartfox-compile-error.png)

解决办法：
- 删除[com.alibaba.smartfox.eclipse.marker]，因为源码中确实没有这个包，[Issues367](https://github.com/alibaba/p3c/issues/367) 
- eclipse默认值添加了[src/main/java]到classpath，并没有添加kotlin，所以需要添加[src/main/kotlin]到classpath。
- 按照上面2步骤操作之后，clean一下应该就ok了


#### 编译
- 执行maven update命令，确保各个依赖项都ok
- 选中[com.alibaba.smartfox.eclipse.plugin]项目，鼠标右键[Configure Kotlin --> Add Kotlin Nature]
- 直接基于smartfox-eclipse目录下的pom文件执行如下命名即可
    ```
    smartfox-eclipse > mvn clean package -Dmaven.test.skip=true
    ```
- 编译完之后，在[\p3c\eclipse-plugin\com.alibaba.smartfox.eclipse.updatesite\target]目录即生成插件的zip包 smartfox-eclipse-plugin.zip。


#### 运行
- 右键单击[com.alibaba.smartfox.eclipse.plugin]项目
- Debug As > Eclipse Application
