---
layout: post
title: 服务器端事件发送SSE
categories: [SpringMVC]
description: 服务器端事件发送
keywords: 异步, sse, SseEmitter
---
## 背景
近期有这么一个需求：
```
手机端需要展示一个比较大的pdf
基于手机端网络/流量/体验等考虑，希望不通过pdf下载然后展示
而是把pdf转成一张张的图片，然后再在手机上展示。
```

## 分析
pdf转图片，肯定是一个比较慢的过程，最好能转完一张就返回一张到前端。  
So，此文要讲的是 请求异步多次返回的技术实现**SSE**  
当然，WebSocket也能做到，它可以双向通信，比SSE(单向发送)强大且复杂，SSE好在比较简单


## 服务器端事件发送 SSE
全称：Server Send Event  
其实严格地说，HTTP 协议无法做到服务器主动推送数据到客户端的。只不过可以变通一下，就是服务器向客户端声明，接下来要发送的是流数据（stream）。  
此时，客户端不会关闭连接，会一直等着服务器发过来的新的数据流。  
SSE 就是利用这种机制，使用流信息向浏览器推送信息。它基于 HTTP 协议，目前除了 IE，其他浏览器都支持。  
IE的话，也可以通过[evensource.js](https://github.com/EventSource/eventsource)来兼容起来。


## 代码实现

### 客户端
需要用到EventSource，并实现onmessage方法
```
if (!!window.EventSource) {
	var source = new EventSource('push');
	s = '';
	source.addEventListener('message', function(e) {
		s += e.data + "<br/>";
		$("#msgFrompPush").html(s);

	});

	source.addEventListener('open', function(e) {
		console.log("连接打开.");
	}, false);

	source.addEventListener('error', function(e) {
		if (e.readyState == EventSource.CLOSED) {
			console.log("连接关闭");
		} else {
			console.log(e);
			source.close();
		}
	}, false);
} else {
	console.log("你的浏览器不支持SSE");
}
```

### 服务端
需要设置类型为event-stream
```
@RequestMapping(value = "/pushV2", produces = "text/event-stream")
	public void pushV2(HttpServletResponse response) {
		response.setContentType("text/event-stream");
		response.setCharacterEncoding("utf-8");
		int count = 0;
		while (true) {
			Random r = new Random();
			try {
				Thread.sleep(1000);
				PrintWriter pw = response.getWriter();
				// 如果浏览器直接关闭，需要check一下
				if (pw.checkError()) {
					System.out.println("客户端主动断开连接");
					return;
				}
				pw.write("data:Testing 1,2,3" + r.nextInt() + "\n\n");
				pw.flush();
				count++;
				if(count>5){
					return;
				}
			} catch (Exception e) {
				e.printStackTrace();
			}
		}
	}
```
以上客户端和服务端的代码示例基于<http://blog.longjiazuo.com/archives/1489>  
做了如下修改：
```
1、原文示例代码中，每个请求只返回了一次数据，服务器每次发完数据断开了连接。
   但SSE默认会自动重连，所以客户端不断地重连（重新发请求）。浏览器F12 network，可以看到刷了很多请求
   这和ajax长轮询没什么区别了。

2、Controller端处理完return返回之后，前端页面会收到一个error事件。浏览器接收到error事件后，SSE又会自动重连，所以我加了一个source.close();
   当然这里close不合理，后面再聊合理的做法
   
```
这里需要知道的是：return之后长连接就断开了，就不是我们想要的持续推送了。  
修改后的代码见Github:  
<https://github.com/yejg/springMvc4.x-project/tree/master/springMvc4.x-serverSendEvent>


## 基于SpringMvc实现
SpringMvc已经对这种异步响应做了很好的封装，我们可以直接返回Callable、DeferredResult或SseEmitter 来更优雅地实现我们的需求。

返回Callable的时候，Spring做了这些事情
- Controller返回一个Callable对象
- Spring MVC开始异步处理并且提交Callable到TaskExecutor在一个单独的线程中进行处理
- DispatcherServlet与所有的Filter的Servlet容器线程退出，但Response仍然开放
- Callable产生结果并且Spring MVC分发请求给Servlet容器继续处理
- DispatcherServlet再次被调用并且继续异步的处理由Callable产生的结果

DeferredResult的处理逻辑和Callable返回差不多，只不过DeferredResult的线程不由SpringMvc管理。  
参考资料： <https://docs.spring.io/spring/docs/4.3.16.RELEASE/spring-framework-reference/html/mvc.html#mvc-ann-async>

Callable和DeferredResult一般用于异步返回单个结果；  
SseEmitter则可以异步多次返回。

在使用SseEmitter写代码前，再解决以下前面提到的一个小问题 -- 合理地close掉EventSource。
```
前面的代码里面，为了避免Controller中return后，浏览器重连，我们直接在error里面把source给close掉了。
source.addEventListener('error', function(e) {
	if (e.readyState == EventSource.CLOSED) {
		console.log("连接关闭");
	} else {
		console.log(e);
		source.close();  // <--- 就是这里
	}
}, false);

SseEmitter有complete()方法，不过执行之后，浏览器也是会收到error事件，并重新请求链接；

那么，最好的做法是：
  Controller处理返回完之后，通知请求端浏览器，告诉它数据都传完了，由浏览器端主动去close掉EventSource。
```

经过上面一系列的分析，可以开始愉快地写代码了：

### 服务端
返回一个自定义的event，type为finish，告知浏览器可以关闭连接了。
```
@RequestMapping("/sseEmitter")
@ResponseBody
public SseEmitter sseEmitterCall() {
	// SseEmitter用于异步返回多个结果，直到调用sseEmitter.complete()结束返回
	SseEmitter sseEmitter = new SseEmitter();
	Thread t = new Thread(new TestRun(sseEmitter));
	t.start();
	return sseEmitter;
}

class TestRun implements Runnable {
	private SseEmitter sseEmitter;
	private int times = 0;

	public TestRun(SseEmitter sseEmitter) {
		this.sseEmitter = sseEmitter;
	}

	@Override
	public void run() {
		while (true) {
			try {
				System.out.println("当前times=" + times);
				sseEmitter.send(System.currentTimeMillis());
				times++;
				Thread.sleep(1000);
				if (times > 4) {
					System.out.println("发送finish事件");
					sseEmitter.send(SseEmitter.event().name("finish").id("6666").data("哈哈"));
					System.out.println("调用complete");
					sseEmitter.complete();
					System.out.println("complete！times=" + times);
					break;
				}
			} catch (IOException | InterruptedException e) {
				e.printStackTrace();
			}
		}
	}
}
```

### 客户端
增加处理finish事件的响应代码
```
if (!!window.EventSource) {
   var source = new EventSource('sseEmitter'); 
   s='';
   source.addEventListener('message', function(e) {
       s+=e.data+"<br/>";
       $("#msgFrompPush").html(s);
   });

   source.addEventListener('open', function(e) {
        console.log("连接打开.");
   }, false);
   
   // 响应finish事件，主动关闭EventSource
   source.addEventListener('finish', function(e) {
       console.log("数据接收完毕，关闭EventSource");
       source.close();
       console.log(e);
  }, false);

   source.addEventListener('error', function(e) {
        if (e.readyState == EventSource.CLOSED) {
           console.log("连接关闭");
        } else {
            console.log(e);
        }
   }, false);
} else {
        console.log("你的浏览器不支持SSE");
}
```
完整代码见：  
<https://github.com/yejg/springMvc4.x-project/tree/master/springMvc4.x-servlet3/src/main>

推荐阅读：  
Server-Sent Events 教程 <http://www.ruanyifeng.com/blog/2017/05/server-sent_events.html>


