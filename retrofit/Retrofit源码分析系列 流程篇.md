---
date : 2018-03-21T20:35:46+08:00
tags : ["Retrofit", "Android", "源码"]
title : "Retrofit源码分析系列 流程篇"

---

本次源码分析是以 GitHub 上 Retrofit 仓库 [2.4.0](https://github.com/square/retrofit/tree/parent-2.4.0)版本来进行的分析，同时以[官方网站](http://square.github.io/retrofit/)文档作为参考。如后续版本有较大改变，恕不另行修改。

# 为什么要写这个系列

Retrofit是当前最火热的Android开源库，是一个十分成熟的网络请求框架。平时用的不亦乐乎，不过真正的源码并没有仔细的读过，这就会导致：如果一段时间不用的话，再用可能会出现使用偏差，毕竟之前只是简单的使用，没有掌握其实现思想。而且前一段时间基于 OKHttp、Retrofit、Rxjava 封装了一个网络请求框架，后来发现写的比较的“死”(不支持 PUT DELETE 操作，不支持多文件上传操作)，不够灵活，打算重新修改一下，正好趁机好好读读源码理解下精髓。闲话少说让我们开始吧。
<!--more-->

# OkHttp 和 Retrofit 有什么区别：

这可能是所有第一次接触 Retrofit 的人都会问的一个问题，包括我。“明明 OkHttp 已经很方便了，这再搞个这玩意儿，那不是自己抢自己生意？” 其实我觉得二者最大区别在于理(zhe)念(xue)。根据Jake大神自己说：
> [OkHttp is purely an HTTP/SPDY client. Retrofit is a high-level REST abstraction on top of HTTP.](https://plus.google.com/112795819945566694222/posts/dkamnxHGkJU)

也就是说Retrofit是一种更高级别的抽象：以请求类型(GET POST DELETE)作为一个参考对象。而 OkHttp 只是一个客户端，以一次次的Http请求作为参考对象，本质上和已被官方抛弃的 HttpClient 是同一类。因此通过 Retrofit 可以更好的进行不同类型的 Http 请求，对于请求的 Url 拼接、参数的传递、传输（Body）/返回类型的转换等等都有更好的支持。

# 介绍

Retrofit 的功能十分强大，总的来说具有以下特性

- 支持 GET, POST, PUT, DELETE, 和 HEAD 等请求
- 支持 Url 操作例如 @Path @Query
- 支持请求体 @Body，如果没有别的 Conventer 那么只能是 RequestBody 类型的参数
- 支持 @Field @Part 和 @MultiPart，如果没有别的 Conventer 那么只能是 RequestBody 类型的 Part 参数
- 支持 Header 操作 *个人感觉这个用处不大毕竟 Header 操作主要都是在 OkHttp 的插值器中用了*
- 支持同步与异步回调 Android 默认同步于主线程
- 支持多种转换器 将 ResponseBody 和 RequestBody 转换为相应的数据结构

## 简单使用

如果之前没有用过 Retrofit，这里我们简单的说明一下，就用官网的例子好了。

1.建立 HTTP API 的接口

```java
public interface GitHubService {
    @GET("users/{user}/repos")
    Call<List<Repo>> listRepos(@Path("user") String user);
}
```

2.使用 Retrofit 生成 API 接口的实现

```java
Retrofit retrofit = new Retrofit.Builder()
    .baseUrl("https://api.github.com/")
    .build();

GitHubService service = retrofit.create(GitHubService.class);
```

*这里偷偷说一句 这样写直接调用是会报错的。。需要加一个GsonConverter，原因后面说明，源码中的Sample是个类似的接口就没事，因为他加了==！*

3.通过 API 接口实现 生成 Call 来与服务端进行通信

```java
Call<List<Repo>> repos = service.listRepos("octocat");
```

这样便得到了一个 Call，类似于 OkHttp3 的 Call 一样，不过这个 Call 是 Retrofit 自己封装的一个接口而已。调用 execute() 或者 enqueue() 之后就可以读取 List 的数据了。

总的来说 Retrofit 有一下的特性：

## 源码结构

[Retrofit](https://github.com/square/retrofit/tree/parent-2.4.0) 主要有5个 model  其中以 retrofit retrofit-adapters retrofit-conventers 为主要代码 model。retrofit-mock 为模拟服务器响应和网络表现不是这次源码分析的重点，samples 是 Retrofit 给出的官方用法。在本系列的的最后一篇，会看这里的示例。
retrofit 结构与主要文件说明如下所示。
```java
    .
    └── retrofit2
        ├── BuiltInConverters.java   //Retrofit 默认的 Conventer.Factory 实现
        ├── CallAdapter.java   //用于对于 Call 的适配，将其转换为需要的类型
        ├── Callback.java    //请求结果的回调与 Call 连用
        ├── Call.java   //Retrofit 对于发送请求和返回结果的封装
        ├── Converter.java    //将 Http 的响应转换为响应类型
        ├── DefaultCallAdapterFactory.java    //Retrofit 默认的 CallAdapter.Factory 实现
        ├── ExecutorCallAdapterFactory.java    //Retrofit 默认的 Conventer.Factory 实现，通过 CallBackExecutor 进行回调
        ├── http   //Http 请求类型及参数注解
        │   ├── Body.java
        │   ├── DELETE.java
        │   ├── Field.java
        │   ├── FieldMap.java
        │   ├── FormUrlEncoded.java
        │   ├── GET.java
        │   ├── Header.java
        │   ├── HeaderMap.java
        │   ├── Headers.java
        │   ├── HEAD.java
        │   ├── HTTP.java
        │   ├── Multipart.java
        │   ├── OPTIONS.java
        │   ├── package-info.java
        │   ├── Part.java
        │   ├── PartMap.java
        │   ├── PATCH.java
        │   ├── Path.java
        │   ├── POST.java
        │   ├── PUT.java
        │   ├── Query.java
        │   ├── QueryMap.java
        │   ├── QueryName.java
        │   ├── Streaming.java
        │   └── Url.java
        ├── HttpException.java
        ├── OkHttpCall.java    //Call 的实现，内部用的 OkHttp 进行的请求
        ├── package-info.java
        ├── ParameterHandler.java   //用于存放 API 调用的参数信息
        ├── Platform.java   //获取当前平台类型
        ├── RequestBuilder.java   //请求构造和 ParameterHandler 连用
        ├── Response.java  
        ├── Retrofit.java  //主体 class
        ├── ServiceMethod.java   //API 方法封装
        └── Utils.java
```

可以看到，Retrofit 的文件其实不多，接下来们就以一次请求来分析其流程。

# 源码分析

## 注解

在使用 Retrofit 来请求之前，我们会先定义每个 API 请求，包括请求方法、请求参数类型、请求响应结构，前两者在 Retrofit 中是以注解形式出现，对应源码中的 http 包下的注解文件。

### 分类

根据 Target 分作用于 `METHOD` 的有 `GET` `POST` `PUT` `DELETE` `HEAD` `HTTP` `OPTIONS` `PATH` 除了 `HTTP` 都是Http协议自带方法。`HTTP` 是可以用自定义方法来进行请求。
作用于`PARAMETER` 的有 `Url` `Path` `Query` `QueryMap` `QueryName` `Field` `FieldMap` `FormUrlEncoded` `Body` `Header` `Headers` `HeaderMap` `Multipart Part` `PartMap` `Streaming`。

### 说明

- `Url` 不在使用定义的 baseUrl 时，使用这个注解，同时意味着这个参数是一个完整的Url 通常放到参数的第一个位置。

- `Path` 用于替换Url中的一部分 替换部分使用{Parm}代替。参数类型使用`stringConvertor`来转换 这个函数我们等下再看。

- `Query` `QueryMap` `QueryName` 都是用来查询 在Url后面进行拼接 Url+?key1=value1&key2=value2 ….. 对于没有 value 的查询可使用`QueryName`

```java
foo.friends("contains(Bob)") ===friends?contains(Bob)
```

- `Field` `FieldMap` 用于 form-encoded 的请求通常和 `FormUrlEncoded` 一起使用,用于表单请求对于 map 要求 k/v 都是非空的。

- `Body` 用于`PUT` `POST`方法传递 request body。如果有对应的 Conventor 则可以转换为相应的 request body 否则只能使用`RequestBody`类型

- `Header` `Headers` `HeaderMap` 都是添加头部请求。不常用。

- `Multipart` `Part` `PartMap` 当 requestBody 是多部分的 使用 `Part` 与 `PartMap` 来修饰参数 同时使用`Multpart` (不能省略) part 的三种使用方法：
    1. `MultipartBody.Part` 类型内容则直接使用  
    2. `RequestBody` 内容的类型直接使用
    3. 其他类型使用Conventor来进行相应的转换类似于body

- `PartMap` 只有后两者方法可以使用 `MultipartBody.Part` 不适用

- `Streaming` 将reponse转换为二进制流  

## Retrofit.Builder

通常在定义好 API 接口之后，我们先会创建一个 Retrofit 的实例。Retrofit 使用了建造者模式，同时 Retrofit 定义为 final，并且其构造函数为包内可见，因此是只能用 Builder 来创建的。

```java
public final class Retrofit {
    private final Map<Method, ServiceMethod<?, ?>> serviceMethodCache = new ConcurrentHashMap<>();

    final okhttp3.Call.Factory callFactory;
    final HttpUrl baseUrl;
    final List<Converter.Factory> converterFactories;
    final List<CallAdapter.Factory> callAdapterFactories;
    final @Nullable Executor callbackExecutor;
    final boolean validateEagerly;

    Retrofit(okhttp3.Call.Factory callFactory, HttpUrl baseUrl,
        List<Converter.Factory> converterFactories, List<CallAdapter.Factory> callAdapterFactories,
      @Nullable Executor callbackExecutor, boolean validateEagerly) {
        ...
    }
    ...
}
```

Retrofit.Builder 的 Constructor 定义如下

```java
public static final class Builder {
    private final Platform platform;
    private @Nullable okhttp3.Call.Factory callFactory;
    private HttpUrl baseUrl;
    private final List<Converter.Factory> converterFactories = new ArrayList<>();
    private final List<CallAdapter.Factory> callAdapterFactories = new ArrayList<>();
    private @Nullable Executor callbackExecutor;
    private boolean validateEagerly;

    Builder(Platform platform) {
      this.platform = platform;
    }

    public Builder() {
      this(Platform.get());
    }

    Builder(Retrofit retrofit) {
      platform = Platform.get();
      callFactory = retrofit.callFactory;
      baseUrl = retrofit.baseUrl;

      converterFactories.addAll(retrofit.converterFactories);
      // Remove the default BuiltInConverters instance added by build().
      converterFactories.remove(0);

      callAdapterFactories.addAll(retrofit.callAdapterFactories);
      // Remove the default, platform-aware call adapter added by build().
      callAdapterFactories.remove(callAdapterFactories.size() - 1);

      callbackExecutor = retrofit.callbackExecutor;
      validateEagerly = retrofit.validateEagerly;
    }
    ...
}
```

二者参数上无太大区别只是 Builder 多了一个 Platform 参数，用于确定使用平台，进而来确定请求完回调在哪个线程。剩下的方法就是各个变量的setter/getter方法。

### 设置baseUrl

这里我们先说一下 uri 的结构：

    [scheme:][//host:port][path][?query][#fragment]

我们知道Url 也是 uri 的一种，因此也是符合 uri 的结构的。因此在设置baseUrl时有三点需要注意：

- `baseUrl` 总是需要`/`结尾。因为 Retrofit 是使用 HttpUrl 来解析的会将其分解为`host`等部分。这样可以使相对路径正确的 append 在`baseUrl`的后面,而后缀 url 开头不要/

- 如果后缀开头如果有`/`则是绝对路径了。会直接 append 到 host 后面而忽略 baseUrl 的 path 部分。

- 如果后缀是一个完全体的 url 则会完全替代 `baseUrl` 注意如果省略Scheme 则默认使用了http 。

### Retrofit实例化

Builder 通过`build`方法生成一个Retrofit的实例。我们来一起看下其源码：

```java
public Retrofit build() {
    if (baseUrl == null) {
        throw new IllegalStateException("Base URL required.");
    }

    okhttp3.Call.Factory callFactory = this.callFactory;
    if (callFactory == null) {
        callFactory = new OkHttpClient();
    }

    Executor callbackExecutor = this.callbackExecutor;
    if (callbackExecutor == null) {
        callbackExecutor = platform.defaultCallbackExecutor();
    }

    // Make a defensive copy of the adapters and add the default Call adapter.
    List<CallAdapter.Factory> callAdapterFactories = new ArrayList<>(this.callAdapterFactories);
    callAdapterFactories.add(platform.defaultCallAdapterFactory(callbackExecutor));

    // Make a defensive copy of the converters.
    List<Converter.Factory> converterFactories = new ArrayList<>(1 + this.converterFactories.size());

    // Add the built-in converter factory first. This prevents overriding its behavior but also
    // ensures correct behavior when using converters that consume all types.
    converterFactories.add(new BuiltInConverters());
    converterFactories.addAll(this.converterFactories);

    return new Retrofit(callFactory, baseUrl, unmodifiableList(converterFactories),
    unmodifiableList(callAdapterFactories), callbackExecutor, validateEagerly);
}
```

函数中会先判断也仅判断 baseUrl 是否存在，其它参数均可以使用默认设置。callFactory 默认`OkHttpClient`，这也就给我们定制 OkHttpClient 的可能，可以实现多种插值器来做多种操作，自定义性非常高。不过这些不在本系列当中，之后会单独出一个封装系列（挖坑了）。之后是通过 platform 生成 callbackExecutor。
我们先看下 Platform 这个类，是一个辅助类，功能很小就是获取当前的运行平台。

```java
class Platform {
  private static final Platform PLATFORM = findPlatform();

  static Platform get() {
    return PLATFORM;
  }

  private static Platform findPlatform() {
    try {
      Class.forName("android.os.Build");
      if (Build.VERSION.SDK_INT != 0) {
        return new Android();
      }
    } catch (ClassNotFoundException ignored) {
    }
    try {
      Class.forName("java.util.Optional");
      return new Java8();
    } catch (ClassNotFoundException ignored) {
    }
    return new Platform();
  }
  ...
```

单例实现，通过类加载的方式来返回相应的平台。根据把不同平台返回相应的 callbackExecutor。Android 平台返回到主线程，callbackExecutor 为 MainThreadExecutor，JVM 则是执行请求的线程，无 callbackExecutor 返回。最后根据 callbackExecutor 生成相应的 defaultCallAdapterFactory，放入 callAdapterFactories。也就是说 `CallAdapterFactory` 是支持返回线程的。最后也是同样添加了默认的 `BuiltInConverters` 到 converterFactories。`CallAdapter.Factory`，`Converter.Factory` 的具体使用，下面再讲。。最后生成了一个Retrofit的实例。

## Retrofit

获取实例之后，就可以开始API请求了。

### 创建API接口

public <T> T create(final Class<T> service)这个函数可以说是整个库最关键的部分了。函数主体是返回了一个 proxy 代理类的实例这样调用 API 接口就相当于调用这个代理类的接口。代理把接口的调用转发给 innvocationHandler ，则 handler 同时可以拿到 API 接口的所有参数和注解。传入到 Invoke 函数，便可以得到 Method 的相关信息。进而得到 ServiceMethod，得到 OkHttpCall 实现了 Call 接口，返回需要的类型。

```java
ServiceMethod<Object, Object> serviceMethod =
    (ServiceMethod<Object, Object>) loadServiceMethod(method);
OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
return serviceMethod.adapt(okHttpCall);
```

## ServiceMethod.Builder

ServiceMethod 就是将请求 API 封装了起来，里面有转换信息，参数信息，url 信息等等。本身只有三个成员方法，同时和 retroft.class 一样，依然使用 Builder 来创建的，所以这里我们就是直接分析 ServiceMethod.Builder 的内容

### ServiceMethod实例化

```java
public ServiceMethod build(){
    // 获取CallAdapter
    callAdapter = createCallAdapter();
    // 获取响应类型
    responseType = callAdapter.responseType();
    ...
    // 获取响应Converter
    responseConverter = createResponseConverter();

    //分析方法注解
    for (Annotation annotation : methodAnnotations) {
        parseMethodAnnotation(annotation);
    }

    //分析参数注解 生成 ParameterHandler
    int parameterCount = parameterAnnotationsArray.length;
    parameterHandlers = new ParameterHandler<?>[parameterCount];
    for (int p = 0; p < parameterCount; p++) {
        ...
        Annotation[] parameterAnnotations = parameterAnnotationsArray[p];
        ...
        parameterHandlers[p] = parseParameter(p, parameterType, parameterAnnotations);
    }
}
//生成实例
return new ServiceMethod<>(this);
```

以上为 build 方法中的有效内容，先获取 CallAdapter，也就是说看看有没有合适的 Adapter 支持接口定义的返回值类型，如果没有就不再执行。其次获取响应类型，并根据响应类型得到相应的 Conventer，如果没有的话就不再执行。
然后将方法注解和参数注解进行解析，获取请求 Url，请求参数信息等。最后生成 ServiceMethod 实例。这里面 parameterAnnotationsArray 被定义为二元数组，methodAnnotations 被定义为一元数组这是为什么？会在第三篇中进行解释。

### 创建 CallAdapter

```java
return (CallAdapter<T, R>) retrofit.callAdapter(returnType, annotations);
```

用来创建“合适”的 adadpter，所谓合适就是返回类型符合，直接调用了 retrofit.callAdapter 方法，跟进去以后发现其最终调用的是 nextCallAdapter 方法，为什么会要再多套一层，直接用 callAdapter 函数里来遍历处理不是挺好吗？这部分内容放到了第三篇。

```java
int start = callAdapterFactories.indexOf(skipPast) + 1;
for (int i = start, count = callAdapterFactories.size(); i < count; i++) {
  CallAdapter<?, ?> adapter = callAdapterFactories.get(i).get(returnType, annotations, this);
  if (adapter != null) {
    return adapter;
    }
}
```

我们知道在 Retrofit 创建的时候，至少是有一个 DefaultCallAdapterFactory，因此我们直接看`DefaultCallAdapterFactory`类。

```java
  @Override
  public CallAdapter<?, ?> get(Type returnType, Annotation[] annotations, Retrofit retrofit) {
    if (getRawType(returnType) != Call.class) {
      return null;
    }

    final Type responseType = Utils.getCallResponseType(returnType);
    return new CallAdapter<Object, Call<?>>() {
      @Override public Type responseType() {
        return responseType;
      }

      @Override public Call<Object> adapt(Call<Object> call) {
        return call;
      }
    };
  }
```

get 方法调用时会判断，返回值的类型，如果不是 Call 类型的话，自己无法处理，直接返回 null ，上层的 CallAdapterFactories 便会寻找下一个可以处理的 Factory。否则会返回一个 CallAdapter，实行转换。CallAdapter 实现了两个方法一个是返回响应类型，另一个是返回转换结果，我们知道Retrofit的create方法最后就是生成一个 `OkHttpCall` 来做的请求，因此 `CallAdapter` 的 adapt 方法直接就返回了参数本身。所以对于 API 请求如果是`Call<R>`类型的请求，其 `CallAdapter` 是不做任何事的。这样便通过合适的 CallAdapterFactory 得到了一个 `CallAdapter`。

### 创建 ResponseConverter

```java
return retrofit.responseBodyConverter(responseType, annotations);
```

与之类似 createResponseConverter 是获得一个“合适”响应转换器。也是直接调用了 retrofit.responseBodyConverter 方法。最后实质调用的是 nextResponseBodyConverter 方法。

```java
int start = converterFactories.indexOf(skipPast) + 1;
    for (int i = start, count = converterFactories.size(); i < count; i++) {
      Converter<ResponseBody, ?> converter =
          converterFactories.get(i).responseBodyConverter(type, annotations, this);
      if (converter != null) {
        //noinspection unchecked
        return (Converter<ResponseBody, T>) converter;
      }
    }
```

同样在创建 Retrofit 实例的时候在 ConventerFactories 中至少有一个`BuiltInConverters`实例。我们来看一下他的 responseBodyConverter 方法

```java
    if (type == ResponseBody.class) {
      return Utils.isAnnotationPresent(annotations, Streaming.class)
          ? StreamingResponseBodyConverter.INSTANCE
          : BufferingResponseBodyConverter.INSTANCE;
    }
    if (type == Void.class) {
      return VoidResponseBodyConverter.INSTANCE;
    }
    return null;
```

方法中也是会针对返回类型来进行判断，如果返回类型是`ResponseBody`或者`Void`才会进行处理否则就是“不合适”。至于 `StreamingResponseBodyConverter`, `VoidResponseBodyConverter` 都是很简单不单独分析了。
因此我们知道了文章开始时用的官网那个例子为什么我会说有问题了，原因就是他用的全是默认配置，那么其API返回类型只能是 Call<ResponseBody>。

### 参数注解解析

方法注解解析很简单就是获取 httpMethod 请求方法（`GET` `POST`等）; hasBody 是否有请求体;判断是否有写死的 `Query` 参数（不允许）; relativeUrl 解析相对url 和 relativeUrlParamNames 里面的参数。如果是别的方法还会解析isFormEncoded isMultipart headers 同时如果没有请求体也不能有 isFormEncoded isMultipart 等等这些判断，很简单，不说了。
参数注解里面有点意思我们看一下：

```java
    } else if (annotation instanceof Path) {
        ...
        gotPath = true;

        Path path = (Path) annotation;
        String name = path.value();
        validatePathName(p, name);

        Converter<?, String> converter = retrofit.stringConverter(type, annotations);
        return new ParameterHandler.Path<>(name, converter, path.encoded());
    } else if ....
    ...
    } else if (annotation instanceof Body) {
        ...
        Converter<?, RequestBody> converter;
        try {
          converter = retrofit.requestBodyConverter(type, annotations, methodAnnotations);
        } catch (RuntimeException e) {
          // Wide exception range because factories are user code.
          throw parameterError(e, p, "Unable to create @Body converter for %s", type);
        }
        gotBody = true;
        return new ParameterHandler.Body<>(converter);
    }
```

在parseParameterAnnotation 函数中将不同的注解分情况讨论，但是仅有`Part` `PartMap` `Body` 是会调用 ConventerFactory 的`requestBodyConverter`函数的其余像`Path` `Query`这些都是 ConventerFactory 的 stringConverter ,因此自定义的 ConventerFactory 通常都是不覆写 stringConverter，同时`BuiltInConverters`也没覆写 stringConverter 而是直接使用自带的`BuiltInConverters.ToStringConverter`来进行的，所以在`Path` 参数这种地方默认都是使用的 String，因此如果自定义 ConventerFactory 时也不建议覆写stringConverter方法，当然了如果要在`Path`参数中使用相应的自定义的 Bean，在 ConventerFactory 就一定要覆写 stringConverter。第三篇有关于这方面的例子。最后生成相应的 ParameterHandler。

## OkHttpCall

回到 innvocationHandler 中，在 ServiceMethod 实例生成后，生成了 OkHttpCall，之前我们说了其实现了 Call 的接口，正在进行通信还是 okhttp3.Call。生成后经过 CallAdapter.adapt 接口转换将合适的类型接口抛出。以默认配置为例，当抛出的API接口进行 execute 或者 enqueue 执行时，便开始正在的网络请求。

### 同步调用

```java
  public Response<T> execute(){
    okhttp3.Call call;
    ...
    call = rawCall;
    if (call == null) {
        try {
          call = rawCall = createRawCall();
        } catch (IOException | RuntimeException | Error e) {
          throwIfFatal(e); //  Do not assign a fatal error to creationFailure.
          creationFailure = e;
          throw e;
        }
    }
    ...
    return parseResponse(call.execute());
  }
```

在执行 execute 的时候会先生成`okhttp3.Call`,调用的是

```java
okhttp3.Call call = serviceMethod.toCall(args);
```

我们知道 serviceMethod 内部有请求方法，请求Url，请求参数等所有信息。因此这里做的是一个信息拼装，然后根据

```java
callFactory.newCall(requestBuilder.build());
```

来获取一个`okhttp3.Call`。调用okhttp3.Call.execute，并将其解析返回。

### 异步调用

enqueue(CallBack) 过程与 execute() 类似，唯一的区别就是在 Retrofit 中定义的 CallBacExecutor 中来进行回调。

### 解析响应

在 enqueue 与 execute 都有`okhttp3.Response`的解析过程。这里的解析就是将 okhttp3.Response 的 body 部分单独提取出来，进行转换，最后再和原始 response 封装起来抛出去，因此这里定义了两个 ResponseBody 来进行解析。可以保证解析完有一个空 body 的 okhttp3.Response，和一个把 ResponseBody转换后的`T`。

```java
  Response<T> parseResponse(okhttp3.Response rawResponse) throws IOException {
    ResponseBody rawBody = rawResponse.body();

    // Remove the body's source (the only stateful object) so we can pass the response along.
    rawResponse = rawResponse.newBuilder()
        .body(new NoContentResponseBody(rawBody.contentType(), rawBody.contentLength()))
        .build();
    ...

    ExceptionCatchingRequestBody catchingBody = new ExceptionCatchingRequestBody(rawBody);
    try {
      T body = serviceMethod.toResponse(catchingBody);
      return Response.success(body, rawResponse);
    } catch (RuntimeException e) {
      // If the underlying source threw an exception, propagate that rather than indicating it was
      // a runtime exception.
      catchingBody.throwIfCaught();
      throw e;
    }
  }
  ...
  R toResponse(ResponseBody body) throws IOException {
    return responseConverter.convert(body);
  }
```

在解析完成后将得到 body, rawResponse 封装称为一个新的 Response 返回，同时 body 就是转换后的值。

这样就是一个完整的 API 请求过程。

# 总结

以上就是 Retrofit 关键部分的源码，从创建实例，解析API，请求数据，转换响应。
![流程总结](/img/retrofit.png)
下一篇，主要精力集中于 CallAdapter 和 Conventer ，会从源码入手分析`RxJavaCallAdapterFactory`与`GsonConverterFactory`。