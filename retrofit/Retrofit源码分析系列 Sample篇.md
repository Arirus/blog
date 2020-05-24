---
date : 2018-03-27T12:55:12+08:00
tags : ["Retrofit", "Android", "源码"]
title : "Retrofit源码分析系列 Sample篇"

---

本次源码分析是以 GitHub 上 Retrofit 仓库 [2.4.0](https://github.com/square/retrofit/tree/parent-2.4.0)版本来进行的分析，同时以[官方网站](http://square.github.io/retrofit/)文档作为参考。如后续版本有较大改变，恕不另行修改。

在第一篇时，我们分析 Retrofit 的请求流程，并在最后给出了请求的流程图，本篇我们就结合着其请求流程，分析下官方给出的几个 Sample。
<!--more-->

# RxJavaObserveOnMainThread

```java
public final class RxJavaObserveOnMainThread {
  public static void main(String... args) {
    Scheduler observeOn = Schedulers.computation();

    Retrofit retrofit = new Retrofit.Builder()
        .baseUrl("http://example.com")
        .addCallAdapterFactory(new ObserveOnMainCallAdapterFactory(observeOn))
        .addCallAdapterFactory(RxJavaCallAdapterFactory.createWithScheduler(io()))
        .build();

    // 这里创建一个Retrofit的实例并加上了两个CallAdapterFactory 一个时ObserveOnMainCallAdapterFactory，还一个是在io线程订阅的RxJavaCallAdapterFactory
  }

  static final class ObserveOnMainCallAdapterFactory extends CallAdapter.Factory {
    final Scheduler scheduler;

    ObserveOnMainCallAdapterFactory(Scheduler scheduler) {
      this.scheduler = scheduler;
    }

    @Override
    public CallAdapter<?, ?> get(Type returnType, Annotation[] annotations, Retrofit retrofit) {
      if (getRawType(returnType) != Observable.class) {
        return null; // 忽视非Observable类型的API.
      }

      // 跳过自己 寻找下一个CallAdapter
      final CallAdapter<Object, Observable<?>> delegate =
          (CallAdapter<Object, Observable<?>>) retrofit.nextCallAdapter(this, returnType,
              annotations);

      return new CallAdapter<Object, Object>() {
        @Override public Object adapt(Call<Object> call) {
          // 获取代理适配的“Call”
          Observable<?> o = delegate.adapt(call);
          // 并使其在给定线程上进行观察
          return o.observeOn(scheduler);
        }

        @Override public Type responseType() {
          return delegate.responseType();
        }
      };
    }
  }
}
```

这里 `RxJavaCallAdapterFactory`实际上就是一个“代理类”，所有的 Call 到 Observable 的转换都是由其来完成，`ObserveOnMainCallAdapterFactory` 只是对返回的 Observable 进行处理，使其在给定的线程上进行观察。在 Factory 的 get 方法中使用了 retrofit.nextCallAdapter 方法直接将自己忽略过，来获取下一个 CallAdapter ，这样就完美的将自己的一些职能转换到另一个 CallAdapter 中，从而获取了一个代理类。同时，这对 retrofit 中的 CallAdapterFacties 提出了一些要求，例如“虚” CallAdapter 要位于“代理” CallAdapter 之前，这样调用 retrofit.nextCallAdapter 才有意义。
最后得到的 Observable 则是订阅于 io 线程观察于 computation 的一个实例，再进行别的操作即可。

# JsonQueryParameters

这个 Sample 中，我们先看其 API 接口定义

```java
  interface Service {
    @GET("/filter")
    Call<ResponseBody> example(@Json @Query("value") Filter value);
  }
```

可以看到，定义了一个 `Get` 请求，不过其 `Query` 的对象是一个非 String 的对象，第一篇的时候我们说过
> 仅有 Part PartMap Body 是会调用 ConventerFactory 的requestBodyConverter函数的其余像 Path Query这些都是 ConventerFactory 的 stringConverter

因此对于 Query 是会默认调用 stringConverter，如果这里不做处理的话，那会直接调用 Filter 基类 Object 的 toString 方法，肯定不是我们想要的结果。因此我们需要根据参数类型进行相应的转换。

```java
  @Retention(RUNTIME)
  @interface Json {
  }

  static class JsonStringConverterFactory extends Converter.Factory {
    private final Converter.Factory delegateFactory;

    JsonStringConverterFactory(Converter.Factory delegateFactory) {
      this.delegateFactory = delegateFactory;
    }

    @Override public Converter<?, String> stringConverter(Type type, Annotation[] annotations,
        Retrofit retrofit) {
      for (Annotation annotation : annotations) {
        if (annotation instanceof Json) {
          // 这里只是举例子因此在构造的时候传入了一个代理Converter.Factory。
          //完全可以和上例一样调用retrofit.nextConventer来获取代理
          Converter<?, RequestBody> delegate =
              delegateFactory.requestBodyConverter(type, annotations, new Annotation[0], retrofit);
          return new DelegateToStringConverter<>(delegate);
        }
      }
      return null;
    }

    static class DelegateToStringConverter<T> implements Converter<T, String> {
      private final Converter<T, RequestBody> delegate;

      DelegateToStringConverter(Converter<T, RequestBody> delegate) {
        this.delegate = delegate;
      }

      @Override public String convert(T value) throws IOException {
        Buffer buffer = new Buffer();
        delegate.convert(value).writeTo(buffer);
        return buffer.readUtf8();
      }
    }
  }
```

这里就定义了一个运行时注解，在 ServiceMethod 解析参数的时候，会获取到 Json 注解，因此可以重写一个 Converter 并实现 stringConverter 方法，将一个 javaBean 转换成合适的 String 用于 Query 参数。
同时也解释了为什么在 ServiceMethod 会有二元数组的参数注解和一元数组的函数注解，因为每个部分的注解可以是多个的，例如这里查询参数就是两个。那么对于参数注解来说就应该是二元的。

# JsonAndXmlConverters

与上一个例子类似，这次API返回值有了多个注解。

```jvav
  interface Service {
    @GET("/") @Json
    Call<User> exampleJson();
    @GET("/") @Xml
    Call<User> exampleXml();
  }
```

因此我们可以定义一个 Conventer 来处理这种情况，因为非参数部分因此只需要实现 responseBodyConverter 函数就可以了

```java
  @Retention(RUNTIME)
  @interface Json {
  }

  @Retention(RUNTIME)
  @interface Xml {
  }

  static class QualifiedTypeConverterFactory extends Converter.Factory {
    private final Converter.Factory jsonFactory;
    private final Converter.Factory xmlFactory;

    QualifiedTypeConverterFactory(Converter.Factory jsonFactory, Converter.Factory xmlFactory) {
      this.jsonFactory = jsonFactory;
      this.xmlFactory = xmlFactory;
    }

    @Override
    public Converter<ResponseBody, ?> responseBodyConverter(Type type, Annotation[] annotations,
        Retrofit retrofit) {
      for (Annotation annotation : annotations) {
        if (annotation instanceof Json) {
          return jsonFactory.responseBodyConverter(type, annotations, retrofit);
        }
        if (annotation instanceof Xml) {
          return xmlFactory.responseBodyConverter(type, annotations, retrofit);
        }
      }
      return null;
    }
  }
```

这样便可以根据不同的注解来使用不同的 Converter 进行转换。
当然如果这个注解是需要在 CallAdadpter 中进行处理也是一样的因为 CallAdapter.Factory.get 方法中，包含了 API 返回的所有注解可以根据注解进行相应的处理。

# 后记

Retrofit 的源码暂时这些内容，里面包含的一些思路还是十分值得学习的，例如API的代理类，反射获取参数类型，CallAdapter，Conventer 的判断创建。只是现在内部是完全使用的 OkHttp 来请求数据，并且没有把接口暴露出来，无法替换造成了一定的死板，不过 OkHttp 依然是很好用的，不换也行。总的来说 Retrofit 是一个十分优秀的库，值得好好读读。
*突然耳机想起了柯南的bgm，卧槽神符合啊！！！*