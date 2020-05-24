---
date : 2018-09-06T09:50:16+08:00
tags : ["OkHttp", "Android", "源码"]
title : "OkHttp源码分析系列 流程篇"

---

本次源码分析是以 GitHub 上 Retrofit 仓库 [3.11](https://github.com/square/okhttp)版本来进行的分析，同时以[官方网站](http://square.github.io/okhttp/)文档作为参考。如后续版本有较大改变，恕不另行修改。

# 为什么要写这个系列

之前我们已经分析了 [Retrofit系列](/post/retrofit源码分析系列-流程篇/)，其实如果只是使用 `Retrofit` 在大多数情况下是完全够用的，而其底层请求也是建立在 `OkHttp` 上的。既然这样，了解一下底层对于我们理解网络请求的流程还是大有裨益的。同时后者引入 `Interceptor` 对于通用的网络请求是非常方便的。因此决定再将 `OkHttp` 好好分析一下。分析流程和之前一样，先从官网给出的例子入手，再到源码，再到自定义部分，好了我们开始。

<!--more-->

# 定位与介绍

不同于 `Retrofit` 的定位，`OkHttp` 是一个纯粹的网络请求工具，大部分东西都需要自己封装，进行请求。不像 `HttpURLConnection` 手动打开链接，你只需要将配置好请求（`request`）使用 `OkhttpClient` 进行请求即可。

## 简单使用

我们依然来使用官网给出的介绍作为例子，进行简单的介绍：

```java
public class GetExample {
  OkHttpClient client = new OkHttpClient(); //生成默认 OkHttpClient 实例

  String run(String url) throws IOException {
    Request request = new Request.Builder()   //生成 Request 实例，默认 GET 方法
        .url(url)
        .build();

    try (Response response = client.newCall(request).execute()) {
      return response.body().string();  //返回 Response
    }
  }

  public static void main(String[] args) throws IOException {
    GetExample example = new GetExample();
    String response = example.run("https://raw.github.com/square/okhttp/master/README.md");
    System.out.println(response);
  }
}

```

以最简单的 `GET` 方法为例，一个基本的网络请求步骤为

    1.生成 `OkHttpClient` 实例，其负责每一次的网络请求。
    2.生成 `Request` 实例，里面包括请求方法，请求参数。
    3.返回 `Response` 实例，可以从里面读取返回的数据。

十分方便，下面我们就分为几部分分别来讨论上述这三个实例。

# OkHttpClient

不像 Retrofit ，必须要使用 Builder 来生成一个相应的实例，OkhttpClient 有共有构造函数来生成一个实例。所有的设置都有默认设置，无需单独指定。
因为像 OkHttpCLient 这样的实例，由于成员变量非常多，而且各个都能配置，所有在类内部实际上还是使用的 Builder 模式，不过构造函数最后是调用了 Builder.build() 来生成了实例：

```java
public Builder() {
  dispatcher = new Dispatcher();
  protocols = DEFAULT_PROTOCOLS;
  connectionSpecs = DEFAULT_CONNECTION_SPECS;
  eventListenerFactory = EventListener.factory(EventListener.NONE); // 请求的回调监听
  proxySelector = ProxySelector.getDefault();
  if (proxySelector == null) {
    proxySelector = new NullProxySelector();
  }
  cookieJar = CookieJar.NO_COOKIES;
  socketFactory = SocketFactory.getDefault();
  hostnameVerifier = OkHostnameVerifier.INSTANCE;
  certificatePinner = CertificatePinner.DEFAULT;
  proxyAuthenticator = Authenticator.NONE;
  authenticator = Authenticator.NONE;
  connectionPool = new ConnectionPool();
  dns = Dns.SYSTEM;
  followSslRedirects = true;
  followRedirects = true;
  retryOnConnectionFailure = true;
  connectTimeout = 10_000;
  readTimeout = 10_000;
  writeTimeout = 10_000;
  pingInterval = 0;
}
```

Builder 的构造函数如上所示，从其命名我们大致知道了他都支持哪些设定。对于 OkhttpClient 有几点需要专门说明一下：OkhttpClient（后面简称 Client） 由于持有连接池和线程池，因此最好是能复用，如果每次请求都单独创建一个 Client 资源消耗会比较大。
也可以通过 Client 的实例调用 newBuilder() 方法来生成一个新的 Client 实例，通过这样生成会和之前的实例共享连接池、线程池和其他配置，同时可以修改单独的配置，这是比较推荐的做法:

```java
OkHttpClient eagerClient = client.newBuilder()
   .readTimeout(500, TimeUnit.MILLISECONDS)
   .build();
Response response = eagerClient.newCall(request).execute();
```

生成了一个新的实例，修改其读取超时设置。
最后就是不用手动关闭线程池和连接池，他们会自动关闭。当然也可以手动关闭，不过要注意，手动关闭后，是不能重新请求的。

其实，Client 里面还有很多需要说明的，这里由于还没有用到，因此我们先放下，后面遇到再说。

# Request

类似 Retrofit ，其本身，没有共有构造函数，因此也是使用 Builder 来生成一个新的实例：

```java
public static class Builder {
  HttpUrl url;
  String method;
  Headers.Builder headers;
  RequestBody body;

  Map<Class<?>, Object> tags : Collections.emptyMap();

  public Builder() {
    this.method = "GET";
    this.headers = new Headers.Builder();
  }
  ...
}
```

Builder 的默认构造函数，默认为 GET 方法，空的 List 来存放 Headers 的内容。可以通过 `method(String method, @Nullable RequestBody body)` 来设定请求的方法，注意方法会对参数有效性进行检查：GET HEAD 方法是不允许传入 RequestBody ，其他方法则必须传入 RequestBody 。其余请求的参数如 URL 等也会进行合法性检查。只会调用 build 方法，便会生成一个 Request 实例。

## RequestBody

上面我们看其，知道 RequestBody 在某些方法请求时是一定要存在的，如 POST 方法。那么我们来看看，其相关源码：

```java
public abstract class RequestBody {
  /** Returns the Content-Type header for this body. */
  public abstract @Nullable MediaType contentType();

  /**
   * Returns the number of bytes that will be written to {@code sink} in a call to {@link #writeTo},
   * or -1 if that count is unknown.
   */
  public long contentLength() throws IOException {
    return -1;
  }

  /** Writes the content of this request to {@code sink}. */
  public abstract void writeTo(BufferedSink sink) throws IOException;

  /**
   * Returns a new request body that transmits {@code content}. If {@code contentType} is non-null
   * and lacks a charset, this will use UTF-8.
   */
  public static RequestBody create(@Nullable MediaType contentType, String content) { ... }

  /** Returns a new request body that transmits {@code content}. */
  public static RequestBody create(final @Nullable MediaType contentType, final ByteString content) { ... }

  /** Returns a new request body that transmits {@code content}. */
  public static RequestBody create(final @Nullable MediaType contentType, final byte[] content) {
    return create(contentType, content, 0, content.length);
  }

  /** Returns a new request body that transmits {@code content}. */
  public static RequestBody create(final @Nullable MediaType contentType, final byte[] content,
      final int offset, final int byteCount) { ... }

  /** Returns a new request body that transmits the content of {@code file}. */
  public static RequestBody create(final @Nullable MediaType contentType, final File file) { ... }
}
```

这里我们先看其构成，屏蔽实现细节。RequestBody 是一个抽象类，三个抽象方法分别是为了：获取body类型，body长度，向流中写入传输的数据。

`void writeTo(BufferedSink sink)` 中 BufferedSink 是 OkIO 的一种缓冲输出流。我们知道原始的 `BufferedOutputStream` 是一种装饰流，必须搭配 `ByteArrayOutputStream` 类似的介质流才能正常使用，因此使用并不方便。OkIO 算是对于 java 流处理的一个升级，BufferedSink 可以写入 byte数组，字符串，数字等，使用更为方便。这里我们不深究其实现细节，知道是个缓冲输出流，可以写入多种数据就行。

最后，还提供了5个静态方法，用于提供 RequestBody 实例。每个方法都需要传入一个 MediaType 实例，用于描述请求体或者响应体的内容类型，详细说明请看[RFC 2045](http://tools.ietf.org/html/rfc2045)。这个类也是没有共有构造函数，需要用 get 或者 parse 将字符串转换为 MediaType 类型。里面使用了正则匹配，先获取了 type 与 subtype，最后获取了字符集，生成了相应的 MediaType。生成 RequestBody 实例，将相应的数据写入到 Sink 中即可。值得一提的是，如果 RequestBody 的 content 是一个 String，同时 MediaType 没规定字符集，则默认使用 Utf-8 字符集。这样一个完整的 RequestBody 救生成了放在 Request 中可以进行请求了。

# Response

好了，炮已架好，炮弹也装填完毕，就剩发射了。我们看上面给出例子：

```Java
Response response = client.newCall(request).execute() ;
```

Client 根据 Request 生成了一个新的 Call ，再进行了执行。Retrofit 中也有一个 Call 接口（为了和上面的相区别，以下检查为RCall），实现为 OkHttpCall ，其实就是对于 Okhttp 的 Call 进行简单的转换，进行请求之类的还是使用 Okhttp 的 Call。这里我们来分析下 Call 的相关内容。

## Call 与 RealCall

```Java
public interface Call extends Cloneable {
  Request request();

  Response execute() throws IOException;

  void enqueue(Callback responseCallback);

  void cancel();

  boolean isExecuted();

  boolean isCanceled();

  Call clone();

  interface Factory {
    Call newCall(Request request);
  }
}
```

Call 使用了工厂模式来产生，主要定义了同步请求方法：execute()；异步请求方法 enqueue(Callback）；状态判断方法 isExecuted()、isCanceled() ；取消方法 cancel()等。

OkHttp 中，Call 的实现只有 RealCall：

```java
final class RealCall implements Call {
  final OkHttpClient client;
  final RetryAndFollowUpInterceptor retryAndFollowUpInterceptor;

  private EventListener eventListener;

  final Request originalRequest;
  final boolean forWebSocket;

  private boolean executed;
  ...
}
```

其中 `forWebSocket` 是针对 WebSocket 协议，不在本系列的讨论范围内。同样 RealCall 没有对外公开并且只有私有构造函数，需要使用静态方法来生成一个新的 RealCall 实例。

## Call.execute()

生成了 Call 的实例，接下来可以进行网络请求了：

```java
@Override public Response execute() throws IOException {
  synchronized (this) {
    if (executed) throw new IllegalStateException("Already Executed");
    executed = true;
  }
  captureCallStackTrace();
  eventListener.callStart(this);
  try {
    client.dispatcher().executed(this);
    Response result = getResponseWithInterceptorChain();
    if (result == null) throw new IOException("Canceled");
    return result;
  } catch (IOException e) {
    eventListener.callFailed(this, e);
    throw e;
  } finally {
    client.dispatcher().finished(this);
  }
}
```

这里面我们可以看到 `getResponseWithInterceptorChain()` 得到了 Response ，因此我们可以知道这个函数肯定是负责了，网络请求的部分，来看下这个函数：

```java
Response getResponseWithInterceptorChain() throws IOException {
  // Build a full stack of interceptors.
  List<Interceptor> interceptors = new ArrayList<>();
  interceptors.addAll(client.interceptors());
  interceptors.add(retryAndFollowUpInterceptor);
  interceptors.add(new BridgeInterceptor(client.cookieJar()));
  interceptors.add(new CacheInterceptor(client.internalCache()));
  interceptors.add(new ConnectInterceptor(client));
  if (!forWebSocket) {
    interceptors.addAll(client.networkInterceptors());
  }
  interceptors.add(new CallServerInterceptor(forWebSocket));

  Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null, null, 0,
      originalRequest, this, eventListener, client.connectTimeoutMillis(),
      client.readTimeoutMillis(), client.writeTimeoutMillis());

  return chain.proceed(originalRequest);
}
```

这里可以看到，全都是对于拦截器的操作，这里按顺序添加的拦截器有：

    Client 自定义的应用拦截器（client.interceptors）；
    失败重试与重定向拦截器（RetryAndFollowUpInterceptor）
    Header 转换拦截器（BridgeInterceptor），请求时默认添加一些header如 User-agent 等，响应体也是类似操作；
    缓存拦截器（CacheInterceptor），从缓存中取出历史响应，或者将网络响应存入。缓存需要提前设置默认不会有默认设置的；
    连接拦截器（ConnectInterceptor），建立网络链接；
    Client 自定义的网络拦截器（client.networkInterceptors）；
    通信拦截器（CallServerInterceptor），最后一个拦截器了，负责和服务器沟通传递数据。

这里也就引出来了，OkHttp 库最精髓的地方--拦截器 Interceptor 。

## Interceptor 与 Chain

在我们解释为啥拦截器如此重要之前，我们先看一下其代码：

```java
public interface Interceptor {
  Response intercept(Chain chain) throws IOException;

  interface Chain {
    Request request();

    Response proceed(Request request) throws IOException;

    @Nullable Connection connection();

    Call call();
    ...
  }
}
```

Interceptor 只是一个接口，里面只有一个函数 intercept(Chain) ，返回 Response。这个 Chain 就连接 `Request` 与 `Response` 的关键。Chain 方法中，我们可以获取 `Request` 的实例，同时可以对其进行修改，对 `Response` 可进行同理的操作。真正的实现了对于每一个拦截器做好自己的工作，不管别人。

> 这样对于把 `Request` 变为 `Response` 这件事来说，每个 Interceptor 都可能完成这件事，所以我们循着链条让每个 Interceptor 自行决定能否完成任务以及怎么完成任务（自力更生或者交给下一个 Interceptor）。

`getResponseWithInterceptorChain()` 中，添加完所有的拦截器，生成了一个 Chain 作为请求的开端。注意这个实例里面，`StreamAllocation` ，`HttpCodec`，`RealConnection` 三个值都是 null，同时 `index`为0，调用了 `Chain.proceed(Request)`，便开始进行链式拦截器请求。

### Chain.proceed(Request)

看源码可知，其实调用了：

    proceed(Request , StreamAllocation , HttpCodec , RealConnection )
然后我们继续往下面看，可以看到这段代码：

```Java
RealInterceptorChain next = new RealInterceptorChain(interceptors, streamAllocation, httpCodec,
    connection, index + 1, request, call, eventListener, connectTimeout, readTimeout,
    writeTimeout);
Interceptor interceptor = interceptors.get(index);
Response response = interceptor.intercept(next);
```

因此，假如当前还有拦截器没有连接在链上，便生成一个新的 Chain 将其调用给相应的拦截器。以此类推，最后可以保证所有拦截器都会在 `Request - Response` 的必经之路上。然而有两个特殊的拦截器，他俩是网络请求的关键。

### ConnectInterceptor

```Java
@Override public Response intercept(Chain chain) throws IOException {
  RealInterceptorChain realChain = (RealInterceptorChain) chain;
  Request request = realChain.request();
  StreamAllocation streamAllocation = realChain.streamAllocation();

  // We need the network to satisfy this request. Possibly for validating a conditional GET.
  boolean doExtensiveHealthChecks = !request.method().equals("GET");
  HttpCodec httpCodec = streamAllocation.newStream(client, chain, doExtensiveHealthChecks);
  RealConnection connection = streamAllocation.connection();

  return realChain.proceed(request, streamAllocation, httpCodec, connection);
}
```

我们看到，`ConnectInterceptor.intercept(Chain)`，方法中最后没有调用 `realChain.proceed(request)`，这是由于之前所有的 Chain 里面的 `StreamAllocation` ，`HttpCodec`，`RealConnection` 均是null，这是进行 Socket 建立连接的地方，因此要传入非空的参数。同时 `StreamAllocation` 在 `RetryAndFollowUpInterceptor` 已经重新赋值了，所以这里是非空的，用于重定向的。`streamAllocation.newStream(client, chain, doExtensiveHealthChecks)` 中便建立 Socket 连接，因此到这里结束和服务器的连接就已经建立了起来了。

### CallServerInterceptor

```java
@Override public Response intercept(Chain chain) throws IOException {
  ...
  realChain.eventListener().requestHeadersStart(realChain.call());
  httpCodec.writeRequestHeaders(request);
  realChain.eventListener().requestHeadersEnd(realChain.call(), request);

  Response.Builder responseBuilder = null;
  if (HttpMethod.permitsRequestBody(request.method()) && request.body() != null) {
      ...
      realChain.eventListener().requestBodyStart(realChain.call());
      long contentLength = request.body().contentLength();
      CountingSink requestBodyOut =
          new CountingSink(httpCodec.createRequestBody(request, contentLength));
      BufferedSink bufferedRequestBody = Okio.buffer(requestBodyOut);

      request.body().writeTo(bufferedRequestBody);
      realChain.eventListener()
          .requestBodyEnd(realChain.call(), requestBodyOut.successfulCount);
      ...
  }

  ...
    responseBuilder = httpCodec.readResponseHeaders(false);  
  ...
    response = response.newBuilder()
        .body(httpCodec.openResponseBody(response))
        .build();
  ...
  return response;
}
```

这里我把 `CallServerInterceptor.intercept(Chain)` 的关键代码提出了。其实就是4步：

    上传 Request 的 header；
    上传 Request 的 requestBody(如果有的话)
    接收 Response的 header；
    接收 Response的 reponseBody(如果有的话)

注意这里，没有再调用`Chain.proceed(request)`方法，因为他就是最后一个，需要他来进行数据访问，最后将解析好的 `Response` 返回个上一个 Chain 就好，每个 Chain 如果对Response 有单独处理，就处理完返回给上一个 Chain ，否则直接返回就可以了。最后在 `RealCall.execute()` 返回 `Reponse` 就完成了一次网络操作。

我们看，这种链式模式其实在 `Retrofit` 也是有体现的，例如在创建 Retrofit 实例时，可能会传入多个 conventer ，其在转换参数类型的时候就是按照链式顺序，如果第一可以转换就算是完成，否则准换到下一个 conventer ，以此类推。

这样一个同步请求便完成。

## Call.enqueue( Callback )

除了同步请求，OkHttp 还支持异步请求。

```java
// Call.enqueue( Callback )
@Override public void enqueue(Callback responseCallback) {
  synchronized (this) {
    if (executed) throw new IllegalStateException("Already Executed");
    executed = true;
  }
  captureCallStackTrace();
  eventListener.callStart(this);
  // 生成一个新的 异步回调，并放入 dispathcher 的队列里
  client.dispatcher().enqueue(new AsyncCall(responseCallback));
}

//Dispatcher.enquene 方法
synchronized void enqueue(AsyncCall call) {
  if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
    runningAsyncCalls.add(call);
    executorService().execute(call);
  } else {
    readyAsyncCalls.add(call);
  }
}
```

意思也好理解，每次将一个回调放入一个队列中，判断当前需不需要启动线程池来进行请求。
由于在 executorService().execute() 要求一个参数是一个 Runnable，因此 Dispatcher 的队列中实际放入的是一个 AsyncCall ，其实现了 Runnable 接口，我们来看看 AsyncCall 实际上是怎样的实现的：

```java
final class AsyncCall extends NamedRunnable {
  private final Callback responseCallback;
  ...

  @Override protected void execute() {
    boolean signalledCallback = false;
    try {
      Response response = getResponseWithInterceptorChain();
      if (retryAndFollowUpInterceptor.isCanceled()) {
        signalledCallback = true;
        responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
      } else {
        signalledCallback = true;
        responseCallback.onResponse(RealCall.this, response);
      }
    } catch (IOException e) {
     ...
    } finally {
      client.dispatcher().finished(this);
    }
  }
}
```

是不是很熟悉，其 execute 方法类似于同步方法，也是直接调用了 getResponseWithInterceptorChain() 方法，进而得到了响应并通过 CallBack 返回了出去。所以同步请求和异步请求在网络请求方法是一样的，不同的是异步请求使用了线程池，同步请求并没有用。

## body

body 就是响应体的部分，通常我们都会需要 body 内容，这里很简单没什么好说的，直接类比 Request 很好理解。

# 小结

OkHttp 的请求流程大致上就是上面的一个过程，最后我们给出整个请求的流程图
![流程总结](/img/Okhttp.png) 下一篇中，我们会着重分析下自定义部分的相关内容，我们下篇见。