---
layout: post
title: 根据Interceptor 分析 OkHttp
categories: Interceptor
description: 根据Interceptor 分析 OkHttp。
keywords: Interceptor, OkHttp
---

根据Interceptor 分析 OkHttp（二）

# 根据Interceptor 分析 OkHttp（二）

标签（空格分隔）： Interceptor OkHttp

---

Interceptor可以说是OkHttp的核心功能，它就是通过Interceptor来完成监控管理、重写和重试请求的。下面是一个简单的Interceptor，可以监控request的输入参数和response的输出内容。

```java
class LoggingInterceptor implements Interceptor {
  @Override public Response intercept(Interceptor.Chain chain) throws IOException {
    Request request = chain.request();

    long t1 = System.nanoTime();
    logger.info(String.format("Sending request %s on %s%n%s",
        request.url(), chain.connection(), request.headers()));

    Response response = chain.proceed(request);

    long t2 = System.nanoTime();
    logger.info(String.format("Received response for %s in %.1fms%n%s",
        response.request().url(), (t2 - t1) / 1e6d, response.headers()));

    return response;
  }
}
```

里面有个方法调用`chain.proceed(request)`，每个Interceptor实现里都有这个调用方法，这个看起来简单的方法却是所有的HTTP请求、生成response的关键所在。

Interceptors可以被串联起来（chained）。OkHttp使用lists来管理Interceptors,让这些Interceptors按顺序被调用。

![Interceptors Diagram](https://raw.githubusercontent.com/wiki/square/okhttp/interceptors@2x.png)

#### Application Interceptors

我们只能通过Application Interceptors或者Network Interceptors来注册自定义的Interceptors，其他Interceptors都是OkHttp帮你做好了的，比如RetryAndFollowUpInterceptor、BridgeInterceptor、CacheInterceptor、ConnectInterceptor、CallServerInterceptor。这里的OkHttp会启动一个拦截器调用链，拦截器递归调用之后最后返回请求的响应Response。这里的拦截器分层的思想就是借鉴的网络里的分层模型的思想。请求从最上面一层到最下一层，响应从最下一层到最上一层，每一层只负责自己的任务，对请求或响应做自己负责的那块的修改。

![Interceptors Diagram](https://ww4.sinaimg.cn/large/006tNbRwly1fdb4w7y0h0j30o90jignd.jpg)

Application Interceptors和Network Interceptors分别位于七层模型的第一层和第六层。这个从`RealCall`里的`getResponseWithInterceptorChain`方法中就可以看出来：

```java
Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    List<Interceptor> interceptors = new ArrayList<>();
    interceptors.addAll(client.interceptors()); // Application Interceptors
    interceptors.add(retryAndFollowUpInterceptor);
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    interceptors.add(new CacheInterceptor(client.internalCache()));
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
      interceptors.addAll(client.networkInterceptors()); // Network Interceptors
    }
    interceptors.add(new CallServerInterceptor(forWebSocket));

    Interceptor.Chain chain = new RealInterceptorChain(
        interceptors, null, null, null, 0, originalRequest);
    return chain.proceed(originalRequest);
  }
```

我们通过这个`LoggingInterceptor` 来说明Application Interceptors和Network Interceptors的区别。

通过`OkHttpClient.Builder`的`addInterceptor()`注册一个 _application_ interceptor:

```java
OkHttpClient client = new OkHttpClient.Builder()
    .addInterceptor(new LoggingInterceptor())
    .build();

Request request = new Request.Builder()
    .url("http://www.publicobject.com/helloworld.txt")
    .header("User-Agent", "OkHttp Example")
    .build();

Response response = client.newCall(request).execute();
response.body().close();
```

URL `http://www.publicobject.com/helloworld.txt`会重定向到`https://publicobject.com/helloworld.txt`,OkHttp会自动[follow](https://www.zybuluo.com/Warning1943/note/698400#follow-up-requests)这次重定向。application interceptor会被调用**once**，并且会返回携带有重定向后的redirected response。

```
INFO: Sending request http://www.publicobject.com/helloworld.txt on null
User-Agent: OkHttp Example

INFO: Received response for https://publicobject.com/helloworld.txt in 1179.7ms
Server: nginx/1.4.6 (Ubuntu)
Content-Type: text/plain
Content-Length: 1759
Connection: keep-alive
```

我们可以看到，会重定向是因我request的URL和response的URL是不同的，日志也打印了两个不同的URLs。

#### Network Interceptors

注册一个Network Interceptors的方式是非常类似的，只需要将`addInterceptor()`替换为`addNetworkInterceptor()`：

```java
OkHttpClient client = new OkHttpClient.Builder()
    .addNetworkInterceptor(new LoggingInterceptor())
    .build();

Request request = new Request.Builder()
    .url("http://www.publicobject.com/helloworld.txt")
    .header("User-Agent", "OkHttp Example")
    .build();

Response response = client.newCall(request).execute();
response.body().close();
```

当我们执行上面这段代码，这个interceptor会执行**twice**。一次是调用在初始的request `http://www.publicobject.com/helloworld.txt`，另外一次是调用在重定向后的redirect request `https://publicobject.com/helloworld.txt`。

```
INFO: Sending request http://www.publicobject.com/helloworld.txt on Connection{www.publicobject.com:80, proxy=DIRECT hostAddress=54.187.32.157 cipherSuite=none protocol=http/1.1}
User-Agent: OkHttp Example
Host: www.publicobject.com
Connection: Keep-Alive
Accept-Encoding: gzip

INFO: Received response for http://www.publicobject.com/helloworld.txt in 115.6ms
Server: nginx/1.4.6 (Ubuntu)
Content-Type: text/html
Content-Length: 193
Connection: keep-alive
Location: https://publicobject.com/helloworld.txt

INFO: Sending request https://publicobject.com/helloworld.txt on Connection{publicobject.com:443, proxy=DIRECT hostAddress=54.187.32.157 cipherSuite=TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA protocol=http/1.1}
User-Agent: OkHttp Example
Host: publicobject.com
Connection: Keep-Alive
Accept-Encoding: gzip

INFO: Received response for https://publicobject.com/helloworld.txt in 80.9ms
Server: nginx/1.4.6 (Ubuntu)
Content-Type: text/plain
Content-Length: 1759
Connection: keep-alive
```

#### Application and Network interceptors 该如何选择

这两个interceptor都有他们各自的优缺点：

**Application Interceptors**

* 不需要关心由重定向、重试请求等造成的中间response产物。
* 总会被调用一次，即使HTTP response是从缓存（cache）中获取到的。
* 关注原始的request，而不关心注入的headers，比如`If-None-Match`。
* interceptor可以被取消调用，不调用`Chain.proceed()`。
* interceptor可以重试和多次调用`Chain.proceed()`。

**Network  Interceptors**

* 可以操作由重定向、重试请求等造成的中间response产物。
* 如果是从缓存中获取cached responses ，导致中断了network，是不会调用这个interceptor的。
* 数据在整个network过程中都可以通过Network  Interceptors监听。
* 可以获取携带了request的`Connection`。

#### 使用Interceptor的说明

在OkHttp 2.2版本才加入了Interceptor功能，而且，Interceptor不能使用`OkUrlFactory`，或者是基于OkHttp的低版本第三方库，比如[Retrofit](http://square.github.io/retrofit/) ≤ 1.8 and [Picasso](http://square.github.io/picasso/) ≤ 2.4 。
