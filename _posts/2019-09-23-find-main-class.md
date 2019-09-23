---
layout: post
title: 运行时找到main方法所在的类
categories: [Java]
description: 运行时找到main方法所在的类
keywords: main
---
### 运行时找到main方法所在的类

```java
private Class<?> deduceMainApplicationClass() {
    try {
        StackTraceElement[] stackTrace = new RuntimeException().getStackTrace();
        for (StackTraceElement stackTraceElement : stackTrace) {
            if ("main".equals(stackTraceElement.getMethodName())) {
                return Class.forName(stackTraceElement.getClassName());
            }
        }
    }
    catch (ClassNotFoundException ex) {
        // Swallow and continue
    }
    return null;
}
```

这个是SpringApplication中获取main class的方法。一直觉得这种方式有点“挫”，^_^，没想到Spring也这么用，不知道还有没有其他更好的办法。








