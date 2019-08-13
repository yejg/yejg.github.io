---
layout: post
title: RequestBody只能读取一次的问题
categories: [SpringMVC]
description: RequestBody只能读取一次的问题
keywords: SpringMVC,RequestBody,读一次
---
#### RequestBody只能读取一次的问题

##### 一、为什么只能读一次
原因很简单：因为是流。想想看，java中的流也是只能读一次，因为读完之后，position就到末尾了。    

##### 二、解决办法
思路：第一次读的时候，把流数据暂存起来。后面需要的时候，直接把暂存的数据返回出去。

实现逻辑：
1. 自定义一个HttpServletRequestWrapper，覆写getInputStream()和getReader()方法。
2. 增加一个Filter，在doFilter()中启用自定义的Wrapper


##### 三、代码
1、自定义的Wrapper

```java
import javax.servlet.ReadListener;
import javax.servlet.ServletInputStream;
import javax.servlet.ServletRequest;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletRequestWrapper;
import java.io.BufferedReader;
import java.io.ByteArrayInputStream;
import java.io.IOException;
import java.io.InputStream;
import java.io.InputStreamReader;
import java.nio.charset.Charset;

public class BodyReaderHttpServletRequestWrapper extends HttpServletRequestWrapper {
    private final byte[] body;

    public BodyReaderHttpServletRequestWrapper(HttpServletRequest request) throws IOException {
        super(request);
        String sessionStream = getBodyString(request);
        body = sessionStream.getBytes(Charset.forName("UTF-8"));
    }

    private String getBodyString(ServletRequest request) throws IOException {
        StringBuilder sb = new StringBuilder();
        InputStream ins = request.getInputStream();

        try (BufferedReader isr = new BufferedReader(new InputStreamReader(ins, Charset.forName("UTF-8")));) {
            String line = "";
            while ((line = isr.readLine()) != null) {
                sb.append(line);
            }
        } catch (IOException e) {
            throw e;
        }
        return sb.toString();
    }

    @Override
    public BufferedReader getReader() throws IOException {
        return new BufferedReader(new InputStreamReader(getInputStream()));
    }

    @Override
    public ServletInputStream getInputStream() throws IOException {

        final ByteArrayInputStream bais = new ByteArrayInputStream(body);

        return new ServletInputStream() {

            @Override
            public int read() throws IOException {
                return bais.read();
            }

            @Override
            public boolean isFinished() {
                return bais.available() == 0;
            }

            @Override
            public boolean isReady() {
                return true;
            }

            @Override
            public void setReadListener(ReadListener readListener) {
            }
        };
    }
}
```

2、加个Filter来启用wrapper

```
import javax.servlet.Filter;
import javax.servlet.FilterChain;
import javax.servlet.FilterConfig;
import javax.servlet.ServletException;
import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;
import javax.servlet.annotation.WebFilter;
import javax.servlet.http.HttpServletRequest;
import java.io.IOException;

// 启用Filter需要在Springboot的启动类加@ServletComponentScan
@WebFilter(filterName = "bodyReaderFilter", urlPatterns = "/*")
public class BodyReaderFilter implements Filter {

    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
    }

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        HttpServletRequest httpServletRequest = (HttpServletRequest) servletRequest;
        ServletRequest requestWrapper = new BodyReaderHttpServletRequestWrapper(httpServletRequest);
        filterChain.doFilter(requestWrapper, servletResponse);
    }

    @Override
    public void destroy() {
    }

}
```


