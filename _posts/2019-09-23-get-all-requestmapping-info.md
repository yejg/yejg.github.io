---
layout: post
title: 获取SpringMVC中所有RequestMapping映射URL信息
categories: [SpringMVC]
description: 获取SpringMVC中所有RequestMapping映射URL信息
keywords: requestMapping,SpringMVC
---
### 获取SpringMVC中所有RequestMapping映射URL信息

SpringMVC启动的时候，会把接口信息收集在RequestMappingHandlerMapping中，故可以通过这个类，拿到全部的映射信息，Sample代码段如下：

```java
@Autowired
private ApplicationContext applicationContext;



Set<String> noLoginUrlSet = new HashSet<>();
RequestMappingHandlerMapping mapping = applicationContext.getBean(RequestMappingHandlerMapping.class);
Map<RequestMappingInfo, HandlerMethod> handlerMethods = mapping.getHandlerMethods();// 就是这个
for (RequestMappingInfo rmi : handlerMethods.keySet()) {
   HandlerMethod handlerMethod = handlerMethods.get(rmi);
   if (handlerMethod.hasMethodAnnotation(NoLogin.class)) {
      PatternsRequestCondition prc = rmi.getPatternsCondition();
      Set<String> patterns = prc.getPatterns();
      noLoginUrlSet.addAll(patterns);
   }
}
```










