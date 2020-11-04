---
layout: post
title: System.currentTimeMillis()会慢吗
categories: [Java]
description: System.currentTimeMillis()会慢吗?还真会
keywords: System, currentTimeMillis
---
### System.currentTimeMillis()会慢吗

最近看一份代码的时候，发现有个打印程序执行耗时的地方，特意写了个类去获取时间。

那为啥不直接用System.currentTimeMillis()呢？好吧，以前没注意这个细节。如果你也不知道，请继续往下看。



#### 现象

先来个demo看看现象

```java
public class CurrentTimeMillisDemo {
    private static final int COUNT = 100;
    public static void main(String[] args) throws Exception {
        long beginTime = System.nanoTime();
        for (int i = 0; i < COUNT; i++) {
            System.currentTimeMillis();
        }
        long elapsedTime = System.nanoTime() - beginTime;
        System.out.println("100 System.currentTimeMillis() serial calls: " + elapsedTime + " ns");

        final CountDownLatch startLatch = new CountDownLatch(1);
        final CountDownLatch endLatch = new CountDownLatch(COUNT);
        for (int i = 0; i < COUNT; i++) {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    try {
                        startLatch.await();
                        System.currentTimeMillis();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    } finally {
                        endLatch.countDown();
                    }
                }
            }).start();
        }
        beginTime = System.nanoTime();
        startLatch.countDown();
        endLatch.await();
        elapsedTime = System.nanoTime() - beginTime;
        System.out.println("100 System.currentTimeMillis() parallel calls: " + elapsedTime + " ns");
    }
}
```

跑一下，输出结果：

>   100 System.currentTimeMillis() serial calls: 5300 ns
>   100 System.currentTimeMillis() parallel calls: 14874400 ns



相差居然这么大，100个并发条件下，耗时是单线程的N倍！



#### 原因

为什么相差这么大？（我也不知道，-_-||）这就是一句简单的获取时间呢

百度谷歌之后，才大概知道原因了。

>   hotspot/src/os/linux/vm/os_linux.cpp文件中，有一个javaTimeMillis()方法，它里面又调用了gettimeofday()，这个是System.currentTimeMillis()的native实现。这个方法里边有3点值得注意的地方：
>
>   -   调用gettimeofday()需要从用户态切换到内核态；
>   -   gettimeofday()的表现受Linux系统计时器（时钟源）影响，在HPET计时器下性能尤其差；
>   -   系统只有一个全局时钟源，高并发或频繁访问会造成严重的争用。

看起来挺有道理的，用户态切换到内核态/原子时钟....那怎么解决呢？



#### 解决

用单个调度线程来按毫秒更新时间戳，相当于维护一个全局缓存。其他线程取时间戳时相当于从内存取，不会再造成时钟资源的争用，代价就是牺牲了一些精确度。源码如下：

```java
public class Clock {

    private final long period;
    private final AtomicLong now;

    private Clock(long period) {
        this.period = period;
        this.now = new AtomicLong(System.currentTimeMillis());
        scheduleClockUpdating();
    }

    public static Clock instance() {
        return InstanceHolder.INSTANCE;
    }

    // 核心是这里，定时去更新时间
    private void scheduleClockUpdating() {
        ScheduledExecutorService scheduler = Executors.newSingleThreadScheduledExecutor(new ThreadFactory() {

            @Override
            public Thread newThread(Runnable runnable) {
                Thread thread = new Thread(runnable, "System Clock");
                thread.setDaemon(true);
                return thread;
            }
        });
        scheduler.scheduleAtFixedRate(new Runnable() {

            @Override
            public void run() {
                now.set(System.currentTimeMillis());
            }
        }, period, period, TimeUnit.MILLISECONDS);
    }

    public long currentTimeMillis() {
        return now.get();
    }

    private static class InstanceHolder {
        public static final Clock INSTANCE = new Clock(1);
    }

}
```

这样写性能差多少呢？ 我的机器上跑了下，差不多省了四分之三，-_-||






















