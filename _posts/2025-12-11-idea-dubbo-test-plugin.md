---
layout: post
title: Dubbo接口测试工具idea插件
categories: [idea-plugin]
description: Dubbo接口测试工具idea插件
keywords: plugin, idea, dubbo
---

# Dubbo接口测试工具idea插件

搞开发的朋友都知道，想试一下别人提供的dubbo接口，还得写点代码，没啥趁手的工具直接调。今天就推荐一款好用的Dubbo测试工具：DubboTest-Y



## 致谢
插件基于[DubboTest](https://github.com/yanglanxing/DubboTest)源码修改而来，先感谢一波~
只是这个开发者好多年都没更新维护了，有几个bug也没人修，实在受不了，我就自己动手了



## 下载
可以在这里[直接下载](/images/posts/idea-dubbo-test-plugin/DubboTest-Y-1.0.zip)
后面我会放到idea插件市场，以方便更多朋友们使用



## 配置使用
在idea中，依次选择 File - Setting - Plugins - ⚙️ - Install Plugin From Disk... ，选择下载的zip包，并按照提示重启idea即可。



不出意外，安装完是这样的效果
![plugin-overview](/images/posts/idea-dubbo-test-plugin/plugin-overview.png)



然后你还需要简单配置一下zk地址，address输入框可以选择以下两种类型

1. 使用zookeeper地址：zookeeper://127.0.0.1:2181
2. 使用dubbo直连：dubbo://127.0.0.1:2288

如下图
![plugin-config](/images/posts/idea-dubbo-test-plugin/plugin-config.png)



最后，选中dubbo接口的某个方法，鼠标右键，选中Run DubboTest-Y，即可
它会自动帮你生成接口对应的参数类型和部分参数，你只需要改成自己需要传的就可以了。



快试试去吧~