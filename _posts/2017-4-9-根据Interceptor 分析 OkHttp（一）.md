---
layout: post
title: 根据Interceptor 分析 OkHttp
categories: Interceptor OkHttp
description: 根据Interceptor 分析 OkHttp。
keywords: Interceptor, OkHttp
---
根据Interceptor 分析 OkHttp
# 根据Interceptor 分析 OkHttp（一）

标签（空格分隔）： Interceptor OkHttp

---
为了更好的阅读体验，可以移步至[根据Interceptor 分析 OkHttp（一）](https://www.zybuluo.com/Warning1943/note/698400#根据interceptor-分析-okhttp一)
在介绍Interceptor前需要理解几个概念


#### Requests
 
每个HTTP请求都包含一个URL，一个method（比如GET/POST），还有一系列的headers。Requests          还可能包含一个body：一个指定content type的data stream。

#### Responses

Responses是通过一个code（比如200代表请求成功、404代表资源未找到），headers还有responses自身的body对Requests的反馈。

#### Rewriting Requests

当你通过OkHttp发送一个HTTP请求的时候，你就在告诉OkHttp“通过这些headers帮我获取到这个URL下的资源”。为了正确和有效，OkHttp需要在发送这个Requests前重写这个Requests。

OkHttp会在原始Requests的基础上添加一些原始Requests缺失的headers，包括Content-Length, Transfer-Encoding, User-Agent, Host, Connection, 以及 Content-Type。如果header里缺失Accept-Encoding，OkHttp会主动添加一个可接收采用gzip压缩response的header，`requestBuilder.header("Accept-Encoding", "gzip");`。另外，如果你已经有cookies，OkHttp会使用这些cookies添加一个cookie header。

#### Rewriting Responses

如果OkHttp在Rewriting Requests中添加了header("Accept-Encoding", "gzip")，并且response有body，OkHttp会将response headers中的Content-Encoding 和 Content-Length去掉，并对response进行解压缩（decompressed）。所以这部分逻辑是：

 1. 开发者没有添加Accept-Encoding时，自动添加Accept-Encoding: gzip
 2. 自动添加的request，response支持自动解压
 3. 手动添加不负责解压缩
 4. 自动解压时移除Content-Length，所以上层Java代码想要contentLength时为-1
 5. 自动解压时移除 Content-Encoding
 6. 自动解压时，如果是分块传输编码，Transfer-Encoding: chunked不受影响。

#### Follow-up Requests

如果你请求的URL资源已经被移除，服务器会返回一个类似于302的状态码来声明新的URL地址。OkHttp会follow这个重定向来获取最终的response。

不过OkHttp支持follow这个重定向的次数是有限制的，最多是20次，如果超过这个次数仍无法获取到最终的response，会抛出一个 ` ProtocolException("Too many follow-up requests: " + followUpCount);`
#### Retrying Requests

有时候connections会失败：也许是因为一个被放进连接池中的 connection 断开连接了，也可能是服务器无法连接。OkHttp会尝试通过其他可用的route来重试请求。

#### Calls

对http的请求封装，属于程序员能够接触的上层高级代码。Calls可以通过两种方式去执行：

 - **Synchronous:**  阻塞线程，直到response返回。
 - **Asynchronous:** 把request放入一个任意thread中的队列，并通过callback在其他线程做回调。

Calls可以在任意thread中被cancel，只要这个request还没有请求完成，request都可以cancel掉。要注意的是，如果request已经被cancel，再做修改request body或者读取response的操作会抛出一个IOException。

#### Dispatch

OkHttp使用Dispatcher作为任务的派发器，有下面这几个关键属性
```
  private int maxRequests = 64;
  private int maxRequestsPerHost = 5;
  
  /** Executes calls. Created lazily. */
  private ExecutorService executorService;

  /** Ready async calls in the order they'll be run. */
  private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();

  /** Running asynchronous calls. Includes canceled calls that haven't finished yet. */
  private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>();

  /** Running synchronous calls. Includes canceled calls that haven't finished yet. */
  private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();
```
如果Call（对http的请求封装）是通过同步的方式发起（Synchronous），则Dispatcher直接将call放入runningSyncCalls队列中，依序进行调用。

如果是通过异步的方式发起（Asynchronous），则Dispatcher需要判断是否立马将该call放入执行队列：
```
    synchronized void enqueue(AsyncCall call) {
  if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
      //添加正在运行的请求
    runningAsyncCalls.add(call);
       //线程池执行请求
    executorService().execute(call);
  } else {
      //添加到缓存队列排队等待
    readyAsyncCalls.add(call);
  }
}
```
其中maxRequests 是最大并发请求数，默认是64个；maxRequestsPerHost 是每个主机最大请求数，默认是5个，这里所说的主机是指Dispatcher会根据call的host来进行归类，相同host的call不能有超过5个在线程池中同时执行，否则要放入等待队列，尽管此时线程池中的并发请求数没有超过默认的64个。这个设计有点类似于服务端的SLB（Server Load Balance），目的是不要让某个主机负载过高，平衡不同host的请求调用。

#### Connections

通过OkHttp请求一个URL的时候，大致过程是这样的：

> 1.OkHttp用这个URL以及配置的`OkHttpClient` 生成`Address`。这个Address声明了如何连接服务器。
> 2.OkHttp会尝试根据这个`Address`从`connection pool`中获取一个`connection`。
> 3.如果在连接池中未找到connection，会通过`RouteSelector`选择一个`Route`生成一个新的`RealConnection`，并将这个新生成的connection放入connection pool。
> 4.通过这个connection发起HTTP request和获取response。

如果一个connection出现了问题，OkHttp会选择一个其他的route进行重试。一旦已经获取到response，这个connection会被放回connection pool以备复用。经过一段时间的休眠后，connection会被从connection pool中移除掉。



 
