---
layout: post
title: 接口标记为@ResponseBody却不进入ResponseBodyAdvice
categories: [SpringMVC]
description: Controller层返回null时不能触发ResponseBodyAdvice#beforeBodyWrite原因分析
keywords: SpringMVC, ResponseBody, ResponseBodyAdvice, beforeBodyWrite
---
### 接口标记为@ResponseBody却不进入ResponseBodyAdvice

#### 一、背景：

我们的接口为了统一，在ResponseBodyAdvice中对返回值做统一处理，默认添加了errorNo和errorInfo字段返回。

最近同事改接口代码的时候，发现接口返回值是空的。乍一看，没什么重大修改。

接口代码大致就是下面这个样子：

```java
@ResponseBody
@RequestMapping("test")
public void test(HttpServletRequest request, HttpServletResponse response){
    // 业务逻辑处理
}
```

#### 二、问题分析

顺着这个接口，单步调试跟到Spring的源码ServletInvocableHandlerMethod#invokeAndHandle方法

```java
public void invokeAndHandle(ServletWebRequest webRequest, ModelAndViewContainer mavContainer, Object... providedArgs) throws Exception {

		Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);
		setResponseStatus(webRequest);

		if (returnValue == null) {
			if (isRequestNotModified(webRequest) || getResponseStatus() != null || mavContainer.isRequestHandled()) {
				mavContainer.setRequestHandled(true);
				// 直接进入到这里就return出去了
				return;
			}
		} else if (StringUtils.hasText(getResponseStatusReason())) {
			mavContainer.setRequestHandled(true);
			return;
		}

		mavContainer.setRequestHandled(false);
		try {
			// 要进到这里才会进入到ResponseBodyAdvice中
			this.returnValueHandlers.handleReturnValue(returnValue, getReturnValueType(returnValue), mavContainer, webRequest);
		} catch (Exception ex) {
			if (logger.isTraceEnabled()) {
				logger.trace(getReturnValueHandlingErrorMessage("Error handling return value", returnValue), ex);
			}
			throw ex;
		}
	}
```

再对着源码，看下return出去的4个条件，核对一下我们的接口

-   void类型，返回值确实是null

-   isRequestNotModified(webRequest)  ,false

-   getResponseStatus() != null ，false

    ​	如果在test方法上标记@ResponseStatus(HttpStatus.OK)，这里取到的就不是null

-   mavContainer.isRequestHandled()，**true**

那么问题来了，mavContainer.isRequestHandled()=true是什么时候设置的呢？

重新debug，发现在ServletResponseMethodArgumentResolver#resolveArgument中设置的，源码截取如下：

```java
public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer, NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {

		if (mavContainer != null) {
            // 就是这里设置的了
			mavContainer.setRequestHandled(true);
		}

		// 略 ...
}
```

#### 三、问题小结

通过上面的分析，问题已经明朗了。

1.  接口一直都是返回null的。开发此次修改的时候，在接口入参里面加了HttpServletResponse
2.  因为有HttpServletResponse入参，就会进入到ServletResponseMethodArgumentResolver
3.  因为ModelAndViewContainer不为null，mavContainer.setRequestHandled(true);
4.  满足前面ServletInvocableHandlerMethod#invokeAndHandle的条件，直接return
5.  返回空结果









