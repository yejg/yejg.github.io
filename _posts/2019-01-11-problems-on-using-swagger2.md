---
layout: post
title: 使用Swagger2时遇到的坑
categories: [Spring]
description: 使用Swagger2时遇到的问题
keywords: Swagger2
---

#### 使用Swagger2时遇到的问题

Swagger2使用起来很简单，加一个@EnableSwagger2注解，并引入如下依赖就ok了

```
<dependency>
	<groupId>io.springfox</groupId>
	<artifactId>springfox-swagger2</artifactId>
	<version>2.7.0</version>
</dependency>
<dependency>
	<groupId>io.springfox</groupId>
	<artifactId>springfox-swagger-ui</artifactId>
	<version>2.7.0</version>
</dependency>
```

配置好之后，启动项目，浏览器输入 http://localhost:8080/swagger-ui.html 应该就能看到api页面了。

But…

##### 问题一：认证

>   Unable to infer base url. This is common when using dynamic servlet registration or when the API is behind an API Gateway. The base url is the root of where all the swagger resources are served. For e.g. if the api is available at http://example.org/api/v2/api-docs then the base url is http://example.org/api/. Please enter the location manually: 

问题原因：项目中使用了Spring Security，swagger2相关的资源没做免登录配置

解决办法：增加如下配置，把Swagger2相关的请求都允许

```
@Component
public class WebSecurityConfigurer extends WebSecurityConfigurerAdapter {
	@Override
	public void configure(WebSecurity web) throws Exception {
		// allow Swagger URL to be accessed without authentication
		web.ignoring().antMatchers(
                "/swagger-ui.html",
                "/v2/api-docs", // swagger api json
                "/swagger-resources/configuration/ui", // 用来获取支持的动作
                "/swagger-resources", // 用来获取api-docs的URI
                "/swagger-resources/configuration/security", // 安全选项
				"/swagger-resources/**"
		);
	}
}
```



##### 问题二：页面提示undifined

解决认证问题之后，打开swagger-ui，页面是出来了，不过页面提示

>   fetching resource list: http://localhost:8080undefined;

api 选择下拉框里面也是undefined，浏览器f12有js报错

>   Uncaught DOMException: Failed to execute 'open' on 'XMLHttpRequest': Invalid URL
>   ​    at h.end (http://localhost:8080/webjars/springfox-swagger-ui/swagger-ui.min.js:13:5344)
>   ​    at u.execute (http://localhost:8080/webjars/springfox-swagger-ui/swagger-ui.min.js:7:29443)
>   ​    at n.exports.c.execute (http://localhost:8080/webjars/springfox-swagger-ui/swagger-ui.min.js:7:27351)
>   ​    at t.exports.g.build (http://localhost:8080/webjars/springfox-swagger-ui/swagger-ui.min.js:7:17807)
>   ​    at t.exports.g.initialize (http://localhost:8080/webjars/springfox-swagger-ui/swagger-ui.min.js:7:16198)
>   ​    at new t.exports (http://localhost:8080/webjars/springfox-swagger-ui/swagger-ui.min.js:7:14664)
>   ​    at C.n.load (http://localhost:8080/webjars/springfox-swagger-ui/swagger-ui.min.js:13:11513)
>   ​    at C.n.updateSwaggerUi (http://localhost:8080/webjars/springfox-swagger-ui/swagger-ui.min.js:13:11182)
>   ​    at C.n.<anonymous> (http://localhost:8080/webjars/springfox-swagger-ui/swagger-ui.min.js:13:10872)
>   ​    at u (http://localhost:8080/webjars/springfox-swagger-ui/lib/backbone-min.js:1:2091)

页面能出来了，至少说明免认证的配置是生效了。

因为swagger的js已经被压缩过，很难从这段js报错发现问题。本想从网上找下源码，无功而返。

不甘心，从网上下载了自定义的ui，稍作整理后，上传到[我的github](https://github.com/yejg/SpringBootExamples/tree/master/spring-boot-swagger/src/main/resources/static)上了

>   这个自定义的有2个版本，只改了UI，没有改java代码逻辑，
>   访问地址分别是 
>
>   -   swagger/index.html
>  -    swagger-v2/docs.html

将这2个地址加入过滤，然后方式试了下，结果发现

>   swagger/index.html 可以正常访问
>
>   swagger-v2/docs.html 也有js报错

有错爆出来，又有源码，就好办了。查看了下js报错的位置，是调用了  /v2/api-docs 后处理返回json数据报错了。

直接在浏览器里面访问了下 /v2/api-docs ，发现回来的数据直接是{“error_no”:“0”,“error_info”:“”}

这里应该不对了，返回的怎么会是这个呢？



debug跟踪一下，springfox.documentation.swagger2.web.Swagger2Controller#getDocumentation方法处理完之后，返回的是

```java
return new ResponseEntity<Json>(jsonSerializer.toJson(swagger), HttpStatus.OK);

// 其中JSON对象代码如下
package springfox.documentation.spring.web.json;
import com.fasterxml.jackson.annotation.JsonRawValue;
import com.fasterxml.jackson.annotation.JsonValue;
public class Json {
  private final String value;
  public Json(String value) {
    this.value = value;
  }
  @JsonValue
  @JsonRawValue
  public String value() {
    return value;
  }
}
```

而且这里的json数据还是正常的，继续往下执行，发现进入到了项目自定义的ResponseBodyAdvice中。在这个增强器中，对返回的数据都统一加了error_no和error_info。

继续debug发现，这个方法里面把body转成了map，而在转换的时候，得到的是空的map！！？？

那为什么明明有数据的body对象，转map就成空的了呢？

```java
// object转map方法核心代码摘录如下
BeanInfo beanInfo = Introspector.getBeanInfo(obj.getClass());
		PropertyDescriptor[] propertyDescriptors = beanInfo.getPropertyDescriptors();
		for (PropertyDescriptor property : propertyDescriptors) {
			String key = property.getName();
			if (StringUtils.equalsIgnoreCase(key, "class")) {
				continue;
			}
			Method getter = property.getReadMethod();
			Object value = getter != null ? getter.invoke(obj) : null;
		}
```

原来，beanInfo.getPropertyDescriptors()获取属性描述的时候，只能拿到 getXXX,setXXX方法。而上面的Json对象的value字段刚好没有getValue方法。