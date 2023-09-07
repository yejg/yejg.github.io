---
layout: post
title: 白嫖一个属于你的私有大模型
categories: [AI]
description: 白嫖一个属于你的私有大模型
keywords: 大模型, big model
---

# 白嫖一个属于你的私有大模型

最近国内的大模型可谓是遍地开花，你瞧瞧：

![image-20230907112613603](/images/posts/ai-big-model/image-20230907112613603.png)

这么火，我也想搞一个试试，于是就有了这篇文章！对，你没看错，就是白嫖。

毕竟人家清华都开源了，哈哈哈hoho~~

先把开源地址贴一下，老铁们可以自行去瞧一瞧：

```
https://github.com/THUDM/ChatGLM-6B
https://huggingface.co/THUDM/chatglm-6b

ChatGLM-6B 是一个开源的、支持中英双语问答的对话语言模型，基于 General Language Model (GLM) 架构，具有 62 亿参数。结合模型量化技术，用户可以在消费级的显卡上进行本地部署（INT4 量化级别下最低只需 6GB 显存）。ChatGLM-6B 使用了和 ChatGLM 相同的技术，针对中文问答和对话进行了优化。经过约 1T 标识符的中英双语训练，辅以监督微调、反馈自助、人类反馈强化学习等技术的加持，62 亿参数的 ChatGLM-6B 已经能生成相当符合人类偏好的回答。
```

最重要的一点，人家遵循Apache-2.0协议。

下面开干吧！



## 准备机器

毕竟是要搭建可以跑起来的环境，机器肯定是必不可少的。好在阿里云有白嫖的使用机器。

1.  进去阿里云免费试用活动页面  https://free.aliyun.com/

2.  申请试用PAI-DSW资源，点击页面上的【立即试用】就可以了。（我因为已经试用了，所以显示的是“已试用”）![image-20230907113838612](/images/posts/ai-big-model/image-20230907113838612.png)

3.  参考试用教程创建PAI平台示例。或者接着往下看

4.  在阿里云页面搜索PAI，点击立即开通，然后进入到PAI控制台。

    开通的时候，有些可选的资源（比如NAS存储等），我因为没有，所以都没选。![image-20230907114333463](/images/posts/ai-big-model/image-20230907114333463.png)

5.  进入控制台后，选择创建DSW实例

    ![image-20230907114642846](/images/posts/ai-big-model/image-20230907114642846.png)

创建的时候，资源选择GPU资源，然后选择 支持资源包抵扣的那款 ecs.gn6v-c8g1.2xlarge

>   如果资源组下拉框是空白的，那么你需要在 上图左侧【工作空间详情】菜单，配置一下计算资源。
>
>   配置的按钮在工作空间详情页面右边【资源管理】，选择public-cluster 即可



镜像选择pytorch1.12，点击创建完成，机器就白嫖好了。



## 下载大模型

前面实例创建完之后，点击【打开】，会进入到机器的web控制台（Data Science Workshop）。

![image-20230907135239571](/images/posts/ai-big-model/image-20230907135239571.png)

在这里，可以在Terminal里面操作了。

1.  先执行安装git相关命令

    >   sudo apt-get update
    >
    >   sudo apt-get install git-lfs

2.  下载模型仓库（因为模型比较大，所以下载下来再执行方便些）

    >   git clone git@hf.co:THUDM/chatglm-6b

3.  下载模型运行代码

    >   git clone https://github.com/THUDM/ChatGLM-6B.git





## 部署启动

### 部署前修改源码

因为我们已经把模型下载下来了，部署前，需要把代码中的模型路径改成你自己的。

比如我们的模型下载在/mnt/workspace/chatglm-6b，我们就需要把 ChatGLM-6B 下的两个文件路径都改一下：

-   cli_demo.py：命令行交互界面
-   web_demo.py：Web图形交互界面

![image-20230907133607674](/images/posts/ai-big-model/image-20230907133607674.png)



### 启动

进入到ChatGLM-6B目录，执行启动命令即可

>   python web_demo.py



命令执行成功，会提示。就表示启动成功了。

>   Running on local URL:  http://127.0.0.1:7860
>
>   To create a public link, set `share=True` in `launch()`.



如果想外网访问，就还需要改一点源码。在web_demo.py文件最末尾，设置share=True

>   #demo.queue().launch(share=False, inbrowser=True)
>
>   demo.queue().launch(share=True, inbrowser=True)

改完后，保存，再次启动，会打印出Running on public URL:XXXX，通过这个链接即可打开体验。



启动中如果遇到缺少一些module，可通过【pip install xxx】进行安装，安装完之后再执行启动命令





## 使用效果

使用效果的话，只能说还有相当长的路需要走。

![image-20230907134339854](/images/posts/ai-big-model/image-20230907134339854.png)

答的是有板有眼的！但是，实际上...

我把完整的代码贴在下面，大家可自行体验评论：

```
import java.util.concurrent.Thread;
import java.util.concurrent.Executors;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors.newSingleThreadExecutor();
import java.util.concurrent.Future;
public class TwoThreads交替打印String {
    public static void main(String[] args) {
        int numThreads = 2;
        String str = "Hello, World!";
        int index = 0;
        ExecutorService executor = Executors.newSingleThreadExecutor();
        List<Future<void>> futures = new ArrayList<>();
        for (int i = 0; i < numThreads; i++) {
            int size = str.length();
            int start = index * size + i;
            int end = start + size;
            int buffer = start * size + size;
            // 打印前一个线程的输出
            futures.add(executor.submit(() -> {
                Thread thread = new Thread(new MyThread(str, buffer));
                thread.start();
            }));
            // 打印当前线程的输出
            futures.add(executor.submit(() -> {
                Thread thread = new Thread(new MyThread(str, buffer));
                thread.start();
            }));
            // 等待当前线程完成
            thread.join();
            // 打印下一个线程的输出
            futures.add(executor.submit(() -> {
                Thread thread = new Thread(new MyThread(str, buffer));
                thread.start();
            }));
            index += size;
        }
        // 打印所有线程的输出
        for (Future<void> future : futures) {
            future.get();
        }
        executor.shutdown();
    }
    private static class MyThread implements Runnable {
        private final String str;
        private final int buffer;
        public MyThread(String str, int buffer) {
            this.str = str;
            this.buffer = buffer;
        }
        @Override
        public void run() {
            for (int i = 0; i < buffer; i++) {
                System.out.print(str.charAt(i) + " ");
            }
        }
    }
}
```



















