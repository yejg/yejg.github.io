---
layout: post
title: Swagger2限定接口范围
categories: [Spring]
description: Swagger2限定接口范围
keywords: Swagger2
---

#### Swagger2限定接口范围

前面在[使用Swagger2时遇到的坑](/2019/01/11/problems-on-using-swagger2)中简单介绍了Swagger的使用。

不过默认情况下，Swagger2会把项目中的所有接口都展示在列表里，特别是你用了Springboot/SpringCloud之后，各种内部health check的接口，但其实这些都没必要展示出来。

这时候，你就需要限定接口的范围了。



##### 实现方法

增加一个配置类，简要代码如下：

```
@Configuration
@EnableSwagger2
public class Swagger2Config {

	@Bean
	public Docket buildDocket() {
		return new Docket(DocumentationType.SWAGGER_2).apiInfo(apiInfo()).select()
		      .apis(RequestHandlerSelectors.basePackage("com.yejg")).paths(PathSelectors.any()).build();
	}
	private ApiInfo apiInfo() {
		return new ApiInfoBuilder().title("接口文档").description("描述文字...").termsOfServiceUrl("https://yejg.top").version("V1.0").build();
	}
}
```

有了这个【RequestHandlerSelectors.basePackage("com.yejg")】限定之后，就只会展示【com.yejg】包下的接口了。so easy…

不过，这时候，有个新问题，如果要增加展示 com.springXXX 的接口，怎么办呢？



##### 优化方案

直接上代码之前，先看下优化方案怎么来的。

从代码看，跟目录范围相关的就是apis方法了，源码如下：

```
// 此Predicate是com.google.common.base.Predicate，不是jdk8的那个，不过原理类似 都是判断的谓词
public ApiSelectorBuilder apis(Predicate<RequestHandler> selector) {
    requestHandlerSelector = and(requestHandlerSelector, selector);
    return this;
}
```

想要支持配置多个目录，就得从这个Predicate下手了。

先看下RequestHandlerSelectors.basePackage是怎么返回Predicate的：

```
  public static Predicate<RequestHandler> basePackage(final String basePackage) {
    return new Predicate<RequestHandler>() {
      @Override
      public boolean apply(RequestHandler input) {
        return declaringClass(input).transform(handlerPackage(basePackage)).or(true);
      }
    };
  }
  
    private static Function<Class<?>, Boolean> handlerPackage(final String basePackage) {
    return new Function<Class<?>, Boolean>() {
      @Override
      public Boolean apply(Class<?> input) {
        return input.getPackage().getName().startsWith(basePackage);
      }
    };
  }
  
```

重点关注上面的handlerPackage方法，它的逻辑就是：判断项目的包路径是否以设定的basePackage开头。

如果改变这里的判断逻辑，判断项目的包路径是否以设定的basePackage1 或者 basePackage2 开头，那就达到同时指定多个目录的效果了。

实现代码如下：

```java
@Configuration
@EnableSwagger2
public class Swagger2Config {

	// 定义分隔符
	private static final String SEPARATOR = ",";

	@Bean
	public Docket buildDocket() {
		return new Docket(DocumentationType.SWAGGER_2).apiInfo(apiInfo()).select()
				.apis(basePackage("com.yejg" + SEPARATOR + "com.springXXX"))
				.paths(PathSelectors.any()).build();
	}

	private ApiInfo apiInfo() {
		return new ApiInfoBuilder().title("接口文档").description("描述文字...")
				.termsOfServiceUrl("https://yejg.top").version("V1.0").build();
	}


	/**
	 * @param basePackage
	 * @return
	 * @see RequestHandlerSelectors#basePackage(String)
	 */
	public static Predicate<RequestHandler> basePackage(final String basePackage) {
		return new Predicate<RequestHandler>() {
			@Override
			public boolean apply(RequestHandler input) {
				return declaringClass(input).transform(handlerPackage(basePackage)).or(true);
			}
		};
	}

	private static Function<Class<?>, Boolean> handlerPackage(final String basePackage) {
		return input -> {
			// 循环判断匹配
			for (String strPackage : basePackage.split(SEPARATOR)) {
				boolean isMatch = input.getPackage().getName().startsWith(strPackage);
				if (isMatch) {
					return true;
				}
			}
			return false;
		};
	}

	private static Optional<? extends Class<?>> declaringClass(RequestHandler input) {
		return Optional.fromNullable(input.declaringClass());
	}

}

```

