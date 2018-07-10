---
layout: post
title: Feign客户端参数传递
categories: [SpringCloud]
description: SpringCloud之feign客户端参数传递用法。看完立马就会撸代码了~
keywords: SpringCloud, feign, 参数
---

- ### 传基本类型参数
使用@RequestParam，注意注解的value参数不可少，代码如下：
```java
@FeignClient(value = "demo-service", fallback = DemoServiceFallback.class) 
public interface DemoService {

    @RequestMapping("/demo-service/test1")
    public String test1(@RequestParam(value = "userName") String userName, @RequestParam(value = "age") int age);
}
```

- ### 传Map传参数
参数太多的时候，上面的方式就要写一堆，可以直接上map <br />
注意，需要加@RequestParam注解，但不需要加注解的value参数

```java
@RequestMapping("/demo-service/test4")
public String test4(@RequestParam Map<String,Object> userMap);
```


- ### 传对象
Fegin传对象的时候，需要加@RequestBody注解，如下：
```java
@RequestMapping("/demo-service/test3")
public String test3(@RequestBody DemoServiceUser user);
```
注意，==服务提供者的Controller的接收参数前也需要加@RequestBody注解==

```java
@RequestMapping("/test3")
public String test3(@RequestBody(required = false) User user, HttpServletRequest request) {
	return "[test3]userName=" + user.getUserName() + ", age=" + user.getAge();
}
```
@RequestBody接收的是一个Json对象的字符串，而不是一个Json对象
如果这时候要使用postman直接请求上面的test3接口，那么需要将Content-Type修改为application/json
![image](https://img-blog.csdn.net/20170315152520135?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYWZ0ZXJsaWZlX3FpeWU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

这样做虽然解决了feign传对象的问题，但是直接请求/test3接口就麻烦了，不能直接使用form-data的形式了。

这里使用RequestInteceptor来解决。[参考资源](https://github.com/spring-cloud/spring-cloud-netflix/issues/1253)


- ### 传对象 + RequestInteceptor
处理思路：feign发请求的时候，将json body转成query。<br />
服务提供者的Controller的接收参数前==不需要@RequestBody注解== <br />
在上例DemoService所在的项目中增加如下代码即可：
```java
@Configuration
public class YryzRequestInterceptor implements RequestInterceptor {
	@Autowired
	private ObjectMapper objectMapper;

	@Override
	public void apply(RequestTemplate template) {
		// feign 不支持 GET 方法传 POJO, json body转query
		if (template.method().equals("GET") && template.body() != null) {
			try {
				JsonNode jsonNode = objectMapper.readTree(template.body());
				template.body(null);

				Map<String, Collection<String>> queries = new HashMap<>();
				buildQuery(jsonNode, "", queries);
				template.queries(queries);
			} catch (IOException e) {
				e.printStackTrace();
			}
		}
	}

	private void buildQuery(JsonNode jsonNode, String path, Map<String, Collection<String>> queries) {
		if (!jsonNode.isContainerNode()) { // 叶子节点
			if (jsonNode.isNull()) {
				return;
			}
			Collection<String> values = queries.get(path);
			if (null == values) {
				values = new ArrayList<>();
				queries.put(path, values);
			}
			values.add(jsonNode.asText());
			return;
		}
		if (jsonNode.isArray()) { // 数组节点
			Iterator<JsonNode> it = jsonNode.elements();
			while (it.hasNext()) {
				buildQuery(it.next(), path, queries);
			}
		} else {
			Iterator<Map.Entry<String, JsonNode>> it = jsonNode.fields();
			while (it.hasNext()) {
				Map.Entry<String, JsonNode> entry = it.next();
				if (StringUtils.hasText(path)) {
					buildQuery(entry.getValue(), path + "." + entry.getKey(), queries);
				} else { // 根节点
					buildQuery(entry.getValue(), entry.getKey(), queries);
				}
			}
		}
	}
}
```


其他参考资料
https://www.jianshu.com/p/7ce46c0ebe9d




