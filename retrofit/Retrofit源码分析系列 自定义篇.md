---
date : 2018-03-26T09:09:57+08:00
tags : ["Retrofit", "Android", "源码"]
title : "Retrofit源码分析系列 自定义篇"

---

本次源码分析是以 GitHub 上 Retrofit 仓库 [2.4.0](https://github.com/square/retrofit/tree/parent-2.4.0)版本来进行的分析，同时以[官方网站](http://square.github.io/retrofit/)文档作为参考。如后续版本有较大改变，恕不另行修改。

# 自定义哪些内容？

一个优秀的框架既要封装良好、简单易用，又要扩展性强、方便修改。上一篇中，我们分析了一次 Http 请求中，Retrofit 的流程是一个什么样的过程，体会到了作为一个网络请求构建的简单易用。这一篇中，我们再来看看 Retrofit 哪些内容允许自定义，方便做出扩展。
<!--more-->
还是从源码看起，通常我们要自定义一些什么，切入点就是 `interface`，因为它允许只暴露接口，而隐藏了相关细节。源码中一共有四个接口 `Call` , `CallBack` , `CallAdapter` , `Conventer`。

- `Call` 的实现类是 `OkHttpCall` , 其内部使用了 okhttp3.Call 用于网络请求，最后被用于传入 CallAdapter 中用于转换 Call 成相应的类型返回出来。
- `CallBack` 用于 `Call` 的 enqueue 函数中，用来返回请求结果，源码中无具体实现类。
- `CallAdapter` 的实现类是 `DefaultCallAdapterFactory` 和 `ExecutorCallAdapterFactory` , 用于将返回响应类型和适配 `Call`。
- `Conventer` 的实现类是 `BuiltInConverters` , 用于转换请求类型和响应类型。

由于 OkHttp 是直接用于 Retrofit 的网络请求，外部并无暴露出接口，同时 Retrofit 的是实例中直接使用的 `okhttp3.Call.Factory` , 所以如果想要修改 `Call` 的实现，需要修改源码，肯定不符合我们的需求。同样, `CallBack` 是直接被使用于 `Call` 当中，因此我们也不对它考虑。最后我们可以自定义的部分就是 `CallAdapter` 和 `Conventer`。

# CallAdapter 

根据上一篇分析，作用总共有两个:

- 返回响应类型，直接决定了 `Conventer` 是否可用于转换
- 将 `Call` 转换成 API 接口返回的类型

当然了，最主要的还是第二个作用。如果 API 的返回类型为一个非泛型类型，那么 API 的返回类型和请求的响应类型则是一致的。

```java
public interface CallAdapter<R, T> {
  /**
   * 响应类型，用于Conventer的转换成Java类型
   */
  Type responseType();

  /**
   * R 为响应类型，T 为返回类型
   */
  T adapt(Call<R> call);
  ...
}
```

而 `CallAdapter` 使用了工厂模式，使用 `CallAdapter.Factory.get(Type returnType, Annotation[] annotations, Retrofit retrofit)` 方法生成一个 `CallAdapter` 实例。

## `RxJavaCallAdapterFactory`

`RxJavaCallAdapterFactory` 是官方提供的 `CallAdapter.Factory` 实现类，用于返回`被观察者`(Single,Completable,Observable等)，对于使用 RxJava 是必不可少的 CallAdapterFactory 。
*官方提供了适配 RxJava 和 RxJava2 的两种适配器。这里我们以`RxJavaCallAdapterFactory`*来作为例子。

```java
public final class RxJavaCallAdapterFactory extends CallAdapter.Factory {
  /**
   * 创建一个实例，不指定订阅线程。默认情况
   */
  public static RxJavaCallAdapterFactory create() {
    return new RxJavaCallAdapterFactory(null, false);
  }

  /**
   * 返回一个创建异步被观察者的实例
   */
  public static RxJavaCallAdapterFactory createAsync() {
    return new RxJavaCallAdapterFactory(null, true);
  }

  /**
   * 指定订阅线程，返回一个创建同步观察者的实例
   */
  @SuppressWarnings("ConstantConditions") // Guarding public API nullability.
  public static RxJavaCallAdapterFactory createWithScheduler(Scheduler scheduler) {
    if (scheduler == null) throw new NullPointerException("scheduler == null");
    return new RxJavaCallAdapterFactory(scheduler, false);
  }
  ...
}
```

`RxJavaCallAdapterFactory` 用三个静态方法来创建的一个实例对象，默认是不指定订阅线程的，createWithScheduler 可以指定订阅线程，在指定线程的同时会返回以一个同步的被观察者，而 createAsync 则可以返回一个异步的被观察者，此时是无法指定订阅线程。

```java
  @Override
  public CallAdapter<?, ?> get(Type returnType, Annotation[] annotations, Retrofit retrofit) {
    Class<?> rawType = getRawType(returnType);
    boolean isSingle = rawType == Single.class;
    boolean isCompletable = rawType == Completable.class;
    //如果不是被观察者返回null
    if (rawType != Observable.class && !isSingle && !isCompletable) {
      return null;
    }

    if (isCompletable) {
      return new RxJavaCallAdapter(Void.class, scheduler, isAsync, false, true, false, true);
    }
    ...
    //判断返回类型
    Type observableType = getParameterUpperBound(0, (ParameterizedType) returnType);
    Class<?> rawObservableType = getRawType(observableType);
    if (rawObservableType == Response.class) {
      ...
      responseType = getParameterUpperBound(0, (ParameterizedType) observableType);
    } else if (rawObservableType == Result.class) {
      ...
      responseType = getParameterUpperBound(0, (ParameterizedType) observableType);
      isResult = true;
    } else {
      responseType = observableType;
      isBody = true;
    }

    return new RxJavaCallAdapter(responseType, scheduler, isAsync, isResult, isBody, isSingle,
        false);
  }
```

Factory 的 get 方法，先判断 API 的返回类型是否是上文提到的三种被观察者类型。是的话表示自己可以处理否则不能。然后根据返回类型判断其响应类型 Response、Result 还是 ResponseBody , 创建响应类型的 RxJavaCallAdapter 。跟其类型检查我们指定其API返回类型支持

- Observable<String>
- Observable<Response<String>>
- Observable<Result<String>>

*这里使用了 Observable 作为被观察者类型， String 作为 ResponseBody 类型。别的类型也是可以的。*

## `RxJavaCallAdapter`

由 `RxJavaCallAdapterFactory` 生成的 CallAdapter 实例。

```java
final class RxJavaCallAdapter<R> implements CallAdapter<R, Object> {
  ...
  @Override
  public Type responseType() {
    return responseType;
  }

  @Override
  public Object adapt(Call<R> call) {
    //创建同步或者异步观察者
    OnSubscribe<Response<R>> callFunc = isAsync
        ? new CallEnqueueOnSubscribe<>(call)
        : new CallExecuteOnSubscribe<>(call);

    //根据返回类型，将观察者进行转换
    OnSubscribe<?> func;
    if (isResult) {
      func = new ResultOnSubscribe<>(callFunc);
    } else if (isBody) {
      func = new BodyOnSubscribe<>(callFunc);
    } else {
      func = callFunc;
    }
    Observable<?> observable = Observable.create(func);

    if (scheduler != null) {
      observable = observable.subscribeOn(scheduler);
    }

    //转换成相应的被观察者
    if (isSingle) {
      return observable.toSingle();
    }
    if (isCompletable) {
      return observable.toCompletable();
    }
    return observable;
  }
  ...
}
```

`responseType()` 返回的响应类型是直接由 RxJavaCallAdapterFactory 传过来的。在 `adapt` 方法中，以从根据同步异步、API返回类型、观察类型创建被观察者并返回。

### `CallExecuteOnSubscribe` `CallEnqueueOnSubscribe`

对于同步执行，创建了 `CallExecuteOnSubscribe`，对于异步执行，则创建了 `CallEnqueueOnSubscribe` ，由于 CallOnSubscribe 实现了 OnSubscribe 方法，因此在被观察者被订阅的时候会直接执行。这里以 CallExecuteOnSubscribe 为例进行分析：

```java
final class CallExecuteOnSubscribe<T> implements OnSubscribe<Response<T>> {
  ...
  @Override
  public void call(Subscriber<? super Response<T>> subscriber) {
    // Since Call is a one-shot type, clone it for each new subscriber.
    Call<T> call = originalCall.clone();
    CallArbiter<T> arbiter = new CallArbiter<>(call, subscriber);
    subscriber.add(arbiter);
    subscriber.setProducer(arbiter);

    Response<T> response;
    try {
      response = call.execute();
    } catch (Throwable t) {
      Exceptions.throwIfFatal(t);
      arbiter.emitError(t);
      return;
    }
    arbiter.emitResponse(response);
  }
}
```

函数中的逻辑比较简单，当订阅时，call 方法被调用，同步执行了 call.execute(), 并将 response 通过调用了 arbiter.emitResponse()。这里的 `CallArbiter`比较难理解，我们只看其关键代码：

```java
final class CallArbiter<T> extends AtomicInteger implements Subscription, Producer {
  ...
  void emitResponse(Response<T> response) {
    while (true) {
      int state = get();
      switch (state) {
        ...
        case STATE_REQUESTED:
          if (compareAndSet(STATE_REQUESTED, STATE_TERMINATED)) {
            deliverResponse(response);
            return;
          }
        ...
      }
    }
  }

  private void deliverResponse(Response<T> response) {
      if (!isUnsubscribed()) {
        subscriber.onNext(response);
      }
    ...
      if (!isUnsubscribed()) {
        subscriber.onCompleted();
      }
    ...
  }
}
```

可以看到 emitResponse 在收到 response 后，直接通过 subscriber.onNext() 方法发射了出去，并且调用了 onCompleted 方法。
CallArbiter 这个类其实实现了 Subscription 这个接口是为了在同步订阅与请求的状态，假如在一个 API 请求的过程中，时间较长，取消订阅了这时会调用 Call.cancel() 方法来取消 API 请求，而实现了 Producer 则是为了处理背压，当下游处理完了事件，才会向上游请求下一个事件。并且继承了 AtomicInteger , 可以执行原子操作，对状态来进行修改。
*RxJava2 中取消了 Observable 处理背压的功能,并且用 Disposable 取代了 RxJava 的 Subscription 的功能因此在 RxJava2 是没有这个 CallArbiter*

### `ResultOnSubscribe` `BodyOnSubscribe`

由于两个 `CallOnSubscribe` 返回的均是 OnSubscribe<Response<R>>，因此如果想改变 Subscriber，需要另一个实现了 OnSubscribe 的对象，这里 ResultOnSubscribe 就分别实现了 OnSubscribe<Result<T>> 和 OnSubscribe<T> 方便后续操作。我们这里分析一下 `BodyOnSubscribe`。

```java
final class BodyOnSubscribe<T> implements OnSubscribe<T> {
  private final OnSubscribe<Response<T>> upstream;

  @Override 
  public void call(Subscriber<? super T> subscriber) {
    upstream.call(new BodySubscriber<T>(subscriber));
  }
  private static class BodySubscriber<R> extends Subscriber<Response<R>> {
    private final Subscriber<? super R> subscriber;

    @Override
    public void onNext(Response<R> response) {
      if (response.isSuccessful()) {
        subscriber.onNext(response.body());
      } else {
        subscriberTerminated = true;
        Throwable t = new HttpException(response);
        ...
        subscriber.onError(t);
        ...
      }
    }
    @Override
    public void onCompleted() {
      if (!subscriberTerminated) {
        subscriber.onCompleted();
      }
    }
  }
}
```

我们可以看到 call 方法直接调用上游的 call 方法并且传入 subscriber，由于上游是 OnSubscribe<Response<T>>，因此 BodySubscriber 实现了 Subscriber<Response<T>>，并在 onNext 当中做出转换将 response 调用了 body 获取了 T 类型的 "requestBody"，通过 subscriber.onNext 发射了出去，相当于做了一个 response -> responseBody 的转换。
`ResultOnSubscribe` 是一样性质，不过是 response -> Result 的转换。

```java
@Override
 public void onNext(Response<R> response) {
  subscriber.onNext(Result.response(response));
}
```

# Conventer

同样在上一篇的时候，我们提到 `Conventer` 的作用也是主要有两个：

- 将默认响应类型 ResponseBody 转换为 T
- 如果是 `POST` 或者 `PUT` 请求参数有 Body 注解时会将T类型转为 RequestBody
- *转化为 String 类型的可以忽略*

```java
public interface Converter<F, T> {
  T convert(F value) throws IOException;


  abstract class Factory {

    public @Nullable Converter<ResponseBody, ?> responseBodyConverter(Type type,
        Annotation[] annotations, Retrofit retrofit) {
      return null;
    }

    public @Nullable Converter<?, RequestBody> requestBodyConverter(Type type,
        Annotation[] parameterAnnotations, Annotation[] methodAnnotations, Retrofit retrofit) {
      return null;
    }

```

同样是用工厂模式生成的各个 conventer

## GsonConverterFactory

类似 `CallAdapter.Factory` ，官方提供了 `GsonConverterFactory` 实现了 `Converter.Factory` ，由于 Gson 对于各个类型有着良好的支持，所以使用这个 Factory 来处理各个不同的类型。

```java
public final class GsonConverterFactory extends Converter.Factory {
  /**
   * 创建一个默认的实例 Gson也是默认配置
   */
  public static GsonConverterFactory create() {
    return create(new Gson());
  }

  /**
   * 使用提供的Gson来创建一个默认的实例。
   * 提供的Gson可以配置很多东西，如指定哪些字段不进行序列化等。
   */
  @SuppressWarnings("ConstantConditions") // Guarding public API nullability.
  public static GsonConverterFactory create(Gson gson) {
    if (gson == null) throw new NullPointerException("gson == null");
    return new GsonConverterFactory(gson);
  }

  private final Gson gson;

  private GsonConverterFactory(Gson gson) {
    this.gson = gson;
  }

  @Override
  public Converter<ResponseBody, ?> responseBodyConverter(Type type, Annotation[] annotations,
      Retrofit retrofit) {
    TypeAdapter<?> adapter = gson.getAdapter(TypeToken.get(type));
    //根据转换类型创建GsonResponseBodyConverter
    return new GsonResponseBodyConverter<>(gson, adapter);
  }

  @Override
  public Converter<?, RequestBody> requestBodyConverter(Type type,
      Annotation[] parameterAnnotations, Annotation[] methodAnnotations, Retrofit retrofit) {
    TypeAdapter<?> adapter = gson.getAdapter(TypeToken.get(type));
    //根据转换类型创建GsonRequestBodyConverter
    return new GsonRequestBodyConverter<>(gson, adapter);
  }
}
```

`GsonRequestBodyConverter`，`GsonResponseBodyConverter`两个分别是请求体转换器和响应体转换器
两个源码都是很简单的，我觉得也没什么需要着重分析的。就是使用 JsonWriter 和 JsonReader 来进行序列化和反序列化将实体类和 Json 之间进行相互转换。

## ScalarsConverterFactory

这个 ConverterFactory 不同于 json 转换器，他是将 String 以及“基本类型”进行转换。

```java
public final class ScalarsConverterFactory extends Converter.Factory {
  ...
  @Override public Converter<?, RequestBody> requestBodyConverter(Type type,
      Annotation[] parameterAnnotations, Annotation[] methodAnnotations, Retrofit retrofit) {
    if (type == String.class
        || type == boolean.class
        || type == Boolean.class
        || type == byte.class
        || type == Byte.class
        || type == char.class
        || type == Character.class
        || type == double.class
        || type == Double.class
        || type == float.class
        || type == Float.class
        || type == int.class
        || type == Integer.class
        || type == long.class
        || type == Long.class
        || type == short.class
        || type == Short.class) {
      return ScalarRequestBodyConverter.INSTANCE;
    }
    return null;
  }
  ...
  @Override
  public Converter<ResponseBody, ?> responseBodyConverter(Type type, Annotation[] annotations,
      Retrofit retrofit) {
    if (type == String.class) {
      return StringResponseBodyConverter.INSTANCE;
    }
    if (type == Boolean.class || type == boolean.class) {
      return BooleanResponseBodyConverter.INSTANCE;
    }
    if (type == Byte.class || type == byte.class) {
      return ByteResponseBodyConverter.INSTANCE;
    }
    if (type == Character.class || type == char.class) {
      return CharacterResponseBodyConverter.INSTANCE;
    }
    if (type == Double.class || type == double.class) {
      return DoubleResponseBodyConverter.INSTANCE;
    }
    if (type == Float.class || type == float.class) {
      return FloatResponseBodyConverter.INSTANCE;
    }
    if (type == Integer.class || type == int.class) {
      return IntegerResponseBodyConverter.INSTANCE;
    }
    if (type == Long.class || type == long.class) {
      return LongResponseBodyConverter.INSTANCE;
    }
    if (type == Short.class || type == short.class) {
      return ShortResponseBodyConverter.INSTANCE;
    }
    return null;
  }
}
```

因此如果有一些特殊需求，例如 API 请求返回的不是 json 格式，而是直接一个 html 网页返回过来，我们可以使用这个 ConventerFactory 使其转换为 String，直接显示就好了。

# 总结

这一篇大致内容就是这样，相比上一篇内容要少一些，也简单一些。我们也看到了 Retrofit 作为一个网络请求框架的可扩展性，下一篇我们来看下源码中给的例子，是一个不错的学习机会。