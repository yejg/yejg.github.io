---
layout: post
title: Eclipse插件改造——EGit源码编译
categories: [eclipse plugin, egit]
description: EGit源码编译
keywords: eclipse, plugin, plug-in, egit
---
## 前言
EGit是基于eclipse的git插件，因最近单位想进一步强化代码规范。  
实现提交代码前强制做代码检查，检查不通过拒绝提交代码到git。所以最近在研究egit，希望能把阿里的p3c结合到一起。   
国内做eclipse插件的太少（插件属于工具类的，来钱少/慢，So...），有技术分享的就更少了。   
断断续续研究了好几天，特意把这些都记录分享出来。

## 1. 源码下载
### 1.1. egit源码
地址：https://github.com/eclipse/egit


### 1.2. jgit源码
EGit是依赖JGit的，所以还需要下载JGit的源码<br />
下载地址：https://github.com/eclipse/jgit


## 2. 环境准备
### 2.1. 下载插件版eclipse
因为是插件开发，所以你需要下载 "Eclipse IDE for Eclipse Committers" 或 "Eclipse for RCP and RAP Developers" <br />
下载地址：https://www.eclipse.org/downloads/packages/

### 2.2. 安装必要的工具
- 依次点击eclipse的【File > Import > Install > Install Software Items from File】菜单   
- 选择源码目录中的【egit\tools\egit-developer-tools.p2f】文件
- 自动安装完成即可   

这一步非常重要，很多依赖的jar都是通过这一步安装的。请确保p2f清单上的依赖项都正确安装成功了。

### 2.3. JDK和eclipse Platform
最低版本要求Java 8.0 ， Eclipse Platform 4.4 (Luna)   
个人建议Eclipse Platform选4.6+

### 2.4. Maven
最低版本要求3.5.2，可在官网下载   
<http://maven.apache.org/download.cgi>   
maven需安装tycho

## 3. 导入
把从git上克隆下来的代码导入到eclipse  
依次点击eclipse的【 Import > Existing Projects into Workspace】即可。  
导入之后，如果有编译错误，请检查前面的[2.2](#2.2)   
**JGit和EGit需要在同级目录**   

在Eclipse中导入EGit和JGit项目后，由于缺少依赖项，它们将无法编译。设置目标平台以解决此问题：   
- 打开EGit中的org.eclipse.egit.target项目
- 选择与Eclipse平台版本匹配的egit-<version>.target文件
- 编辑器中，单击右上角的Set as Target Platform链接（这个过程可能需要一段时间，因为它会下载各种依赖项）
之后，工作区应该干净利落地构建。如果没有，请尝试Project> Clean ...> All。
- 如果这些操作都不行，请尝试点击[egit- <version>.target]右上角的[reload target platform]   

另外，通过【Preference -> Plug-in development -> Target Platform】也可以选的目标平台。 

因为网络问题(GFW?)，你选择的target platform所需的jar很可能下载不下来。这时候我建议你直接选择[Running platform]

## 4. 运行
- 右键单击org.eclipse.egit.ui项目
- Debug As > Eclipse Application

## 5. 编译打包
编译过程略复杂，因为涉及到jgit和egit的编译。   
编译依赖环境配置如下：
- JGit使用JDK、Maven3.5.2+编译
- EGit使用Maven和Tycho。[关于Tycho](http://holbrook.github.io/2014/01/08/build_osgi_bundle_with_tycho_maven_plugin.html)


由于Tycho限制，不能在同一个反应器构建中混合使用pom和manifest构建，因此pom JGit构建必须在构建manifest JGit打包项目之前单独运行。且3个版本编译必须共享相同的本地maven库。

编译命令如下(按顺序依次执行)：

```bat
F:\Git_OpenSource\jgit>mvn clean install  --settings E:\apache-maven-3.5.4\conf\cairenhui-settings-jdk1.8.xml -Dmaven.test.skip=true

F:\Git_OpenSource\jgit>mvn -f org.eclipse.jgit.packaging/pom.xml clean install  --settings E:\apache-maven-3.5.4\conf\cairenhui-settings-jdk1.8.xml -Dmaven.test.skip=true

F:\Git_OpenSource\egit>mvn clean install  --settings E:\apache-maven-3.5.4\conf\cairenhui-settings-jdk1.8.xml -Dmaven.test.skip=true
```
以上命名执行完之后，生成的文件在F:\Git_OpenSource\egit\org.eclipse.egit.repository\target\org.eclipse.egit.repository-5.0.2-SNAPSHOT.zip

至此，源码打包编译就完成了。   
后面再分享改造源码实现目标的细节。

