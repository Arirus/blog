---
date : 2018-09-10T13:30:45+08:00
tags : ["OkHttp", "Android", "源码"]
title : "OkHttp源码分析系列 自定义篇"

---

本次源码分析是以 GitHub 上 Retrofit 仓库 [3.11](https://github.com/square/okhttp)版本来进行的分析，同时以[官方网站](http://square.github.io/okhttp/)文档作为参考。如后续版本有较大改变，恕不另行修改。

# 自定义哪些内容？

上一篇中，我们分析了 OkHttp 在进行网络请求时候的，一个大致原理。发现其比较简单的，不像 Retrofit 比较复杂，简单来说就是 `生成请求`，`请求调用`，`返回响应`。因此自定义内容有限，主要还是以`拦截器 interceptor` 为主，本篇我们就以 `HttpLoggingInterceptor` 来分析下，如果要自定义拦截器需要注意哪些方面。

<!--more-->

# ApplicationInterceptor NetworkInterceptor

在分析 `HttpLoggingInterceptor` 之前，我们先来看看，这两个有啥区别。
上一篇中，我们在分析 getResponseWithInterceptorChain() 其实有说过，二者的差别在于 `调用先后顺序` 的不同。ApplicationInterceptor 在所有自带的拦截器调用之前调用，而 NetworkInterceptor 在 ConnectInterceptor 调用后（socket 连接建立之后），CallServerInterceptor 调用之前（请求数据之前）调用。这是二者唯一的差别。

我们给出一个例子来进行说明：

```java
  OkHttpClient client = new OkHttpClient.Builder()
      .addInterceptor(new Interceptor() {
        @Override public Response intercept(Chain chain) throws IOException {
          System.out.println("Request Chain url: "+chain.request().url());
          Response response = chain.proceed(chain.request());
          System.out.println("Response Chain url: "+response.request().url()+" isRedirect: "+response.isRedirect());
          return response;
        }
      })
      .build();

  String run(String url) throws IOException {
    Request request = new Request.Builder()
        .url(url)
        .build();
    try (Response response = client.newCall(request).execute()) {
      return String.valueOf(response.request().url());
    }
  }

  public static void main(String[] args) throws IOException {
    GetExample example = new GetExample();
    String response = example.run("http://google.com");
    System.out.println("final url: "+response);
  }
```

代码很简单就是生成一个客户端，然后对给出的 url 调用 GET 方法，将收到的 Response 中 Request 部分的 Url 打印出来。Interceptor 部分也是很简单，对于请求没做任何处理，只是打印出来其 Url 并打印出响应体中，是否是重定向的。可以得到下面的结果：

```java
Request Chain url: http://google.com/
Response Chain url: http://www.google.com/ isRedirect: false
final url: http://www.google.com/
```

我们在请求的时候传入的是一个 http 网址，网站解析后将其自动重定向到 https 的 url，因此最后的 GET 方法是使用的 https 的 url 来进行请求，同时最终返回的响应也是重定向后的网址。

假如我们换成 addNetworkInterceptor 再进行请求，会得到如下的结果

```java
Request Chain url: http://google.com/
Response Chain url: http://google.com/ isRedirect: true
Request Chain url: http://www.google.com/
Response Chain url: http://www.google.com/ isRedirect: false
final url: http://www.google.com/
```

我们可以看到，如果是使用了 NetworkInterceptor 则会将每一次 socket 连接和服务端通信的情况展现出来，而不是仅仅展现最后一次返回值。

大致上，我们也明白二者是怎么样的区别了。因此可以简单的给两类拦截器下个结论：

    ApplicationInterceptor 在一次请求的过程中只会调用一次; NeteorkInterceptor 可能调用不止一次。
    ApplicationInterceptor 请求的时候不需要考虑是否重定向，根据其 reponse 读出 isRedirect 一定是 false。
    当有缓存的时候，ApplicationInterceptor 一样会调用，但是 NetworkInteceptor 则不会被调用。

# 自定义拦截器

## 头部拦截器

有了上面的经验，我们可以开始自己来编写拦截器了。现在考虑一种需求：Api 请求的时候大多数的时候都要带上 Token 来进行验证，如果每次请求的话都将其以参数的方式来构造不仅麻烦还容易出错，但配上拦截器的话就会简单很多。

```java
private static class HeadInterceptor implements Interceptor{
  @Override public Response intercept(Chain chain) throws IOException {
    Request.Builder newReq = chain.request().newBuilder();
    newReq.header("Authorization","Bearer "+TokenWrapper.INSTANCE.getToken());
    return chain.proceed(newReq.build());
  }
}

OkHttpClient client = new OkHttpClient.Builder()
    .addInterceptor(new HeadInterceptor())
    .addInterceptor(interceptor)
    .build();

example.run("http://httpbin.org/get").body().print()

log:
--> GET http://httpbin.org/get
Authorization: Bearer XXXXXXXXXXXX
--> END GET
<-- 200 OK http://httpbin.org/get (776ms)
...
<-- END HTTP
final url: {
  "args": {},
  "headers": {
    "Accept-Encoding": "gzip",
    "Authorization": "Bearer XXXXXXXXXXX",
    "Connection": "close",
    "Host": "httpbin.org",
    "User-Agent": "okhttp/3.12.0-SNAPSHOT"
  },
  "url": "http://httpbin.org/get"
}
```

我们把一个头部拦截器实例传入到 OkHttp ,在每次请求的时候，都将 `Token` 写入到 header 里面。我们根据 logger 拦截器看到，自定义传入的 header 只有 `Authorization` 。返回的 ResponseBody 也可以看到服务器也确实是收到了我们的 `Token`。

## 参数拦截器

在讲参数拦截器之前，我们先来看看FormBody MultipartBody两个，库自带 RequestBody 的实现。
上一篇中，我们看了 RequestBody 的源码，其中包括 MediaType 和 content，这两个部分。
前者表面数据类型，后者表明数据内容。而使用 FormBody 可以便利的生成表单的 RequestBody，就像使用map 一样，调用 add 方法来传入需要的键值对。相反，如果直接使用 RequestBody.create() 反而不是很方便。

```java
RequestBody formBody = new FormBody.Builder().add("user", "Arirus")
    .add("passwd","qqqqqq").build();
RequestBody requestBody =
    RequestBody.create(MediaType.get("application/x-www-form-urlencoded"), "user=Arirus&passwd=qqqqqq");
Request request =
    new Request.Builder().url("http://httpbin.org/post").post(requestBody).build();

log:
{
  ...
  "form": {
    "passwd": "qqqqqq",
    "user": "Arirus"
  }
  ...
}
```

二者都可以正常提交表单，但是明显第一种方式更简单一些。当然除了这个之外还有另一个重要的原因：

    RequestBody.create() 传入 content 默认都是经过 url-encode 的
这个可能不好理解，我举个例子：假如 `passwd` 传入的值为 `qqq+qqq%6D`,分别运行下两种请求，结果如下。

```java
// requestBody 响应
"form": {
  "passwd": "qqq qqqm",
  "user": "Arirus"
},
// formBody 响应
"form": {
  "passwd": "qqq+qqq%6D",
  "user": "Arirus"
},
```

我们发现使用 `requestBody` 提交表单，数据好像是被转义了一样，而 `formBody` 提交表单没有这个问题。
这就是上面原因导致的结果：`+` 会被转义为一个空格，`%HH` 会被转义成一个字符。所以推荐使用 `requestBody` 来提交表单数据。同样对于`MultipartBody`，如果手动传入数据会比较复杂，建议使用其进行数据传递。

说完上面这些，我们来分析下如何对于传递参数`动手脚`：

```java
@Override public Response intercept(Chain chain) throws IOException {
  System.out.println(chain.request().body());
  if (!(chain.request().body() instanceof FormBody)) return chain.proceed(chain.request());

  for (int i = 0; i < ((FormBody) chain.request().body()).size(); i++) {
    System.out.printf("Key:%s Value:%s \n",
        ((FormBody) Objects.requireNonNull(chain.request().body())).name(i),
        ((FormBody) Objects.requireNonNull(chain.request().body())).value(i));
  }

  return chain.proceed(chain.request());
}
log：
Key:user Value:Arirus
Key:passwd Value:qqq+qqq%6D
```

这样我们发现如果 RequestBody 是 FormBody 的实例，就可以遍历键值对的 list，获取所有参数。对于 MultipartBody 则是相同原理，其内部有 Part 的 List，可以遍历得到每个 Part。

# 小结

本篇当中主要是分析了，两类拦截器的区别，同时分析使用 FormBody 和 MultiPartBody 来生成 RequestBody 的优点，最后分别给出一个头部拦截器和参数拦截器的例子。相对于 Retrofit，OkHttp 更简单一些，那么这个系列文章就到这里结束了，有缘再见。
