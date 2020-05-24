---
date : 2018-07-17T16:55:06+08:00
tags : ["Rxjava", "Android", "用法"]
title : "Rxjava朝花夕拾 Rxjava2.X"

---

本次源码分析是以 GitHub 上 RxJava 仓库 [2.1.1](https://github.com/ReactiveX/RxJava)版本来进行的分析，同时以[官方中文文档](https://legacy.gitbook.com/book/mcxiaoke/rxdocs/details)作为参考。

之前的两篇操作符文章，我们都是基于 Rxjava 1.X 来进行分析的，而 1.X 版本已经在今年三月底时，停止了维护。今后新项目主要是都是基于 Rxjava 2.X 来进行的开发，因此本篇我们来看看相对于老版本，新版本有哪些改变。

<!--more-->

# Reactive Streams

2.0 版本完全是基于响应流的规范来进行设计的，作为设计的基石，有必要在这里着重介绍。
Java 的响应流只包含4个接口：

```java
public interface Publisher<T> {
    public void subscribe(Subscriber<? super T> s);
}

public interface Subscriber<T> {
    public void onSubscribe(Subscription s);
    public void onNext(T t);
    public void onError(Throwable t);
    public void onComplete();
}

public interface Subscription {
    public void request(long n);
    public void cancel();
}

public interface Processor<T,R> extends Subscriber<T>, Publisher<R> {
}
```

> 发布者（publisher）是潜在无限数量的有序元素的生产者。它根据收到的要求向当前订阅者发布（或发送）元素。

> 订阅者（subscriber）从发布者那里订阅并接收元素。发布者向订阅者发送订阅令牌（subscription token）。使用订阅令牌，订阅者从发布者哪里请求多个元素。当元素准备就绪时，发布者向订阅者发送多个或更少的元素。订阅者可以请求更多的元素。发布者可能有多个来自订阅者的元素待处理请求。

> 订阅（subscription）表示订阅者订阅的一个发布者的令牌。当订阅请求成功时，发布者将其传递给订阅者。订阅者使用订阅令牌与发布者进行交互，例如请求更多的元素或取消订阅。

> 处理者（processor）充当订阅者和发布者的处理阶段。Processor接口继承了Publisher和Subscriber接口。它用于转换发布者——订阅者管道中的元素。

因此我们看出来，Publisher 对应1.X 的 Observable，Subscriber 对应 Observer，Processor 对应 Subject。因此 2.X 在开发思路上和上一个版本是如出一辙，更多的一些细节的打磨。

# Observable 和 Flowable

在 RxJava 2.0 中 增加了被观察者的新实现 Flowable 来支持背压 Backpressure，同时 Observable 不再支持背压。新的 Observable 在创建时也与老版本有所不同。

```java
Observable.create(new ObservableOnSubscribe<Integer>() {
  @Override
  public void subscribe(ObservableEmitter<Integer> e) throws Exception {
    e.onNext(1);
  }
}).subscribe();

Flowable.create(new FlowableOnSubscribe<Integer>() {
  @Override
  public void subscribe(FlowableEmitter<Integer> e) throws Exception {
    e.onNext(1);
  }
}, BackpressureStrategy.ERROR).subscribe();
```

以此引出来

    Observable(被观察者)/Observer（观察者）
    Flowable(被观察者)/Subscriber(观察者)
两种观察者模式。

对比下两种观察者模式：

```java
// Observable Observer (这里只用了一个 Consumer 来接收参数)
Observable.interval(1, TimeUnit.MILLISECONDS)
    .observeOn(Schedulers.newThread())
    .subscribe(aLong -> {
      try {
        Thread.sleep(5000);
      } catch (InterruptedException e) {
        e.printStackTrace();
      }
      Log.w("TAG","---->"+aLong);
    });

// Flowable Subscriber (这里只用了一个 Consumer 来接收参数)
Flowable.interval(1, TimeUnit.MILLISECONDS)
    .observeOn(Schedulers.newThread())
    .subscribe(aLong -> {
      try {
        Thread.sleep(5000);
      } catch (InterruptedException e) {
        e.printStackTrace();
      }
      Log.w("TAG","---->"+aLong);
    });
```

对比发现，使用 Observable/Observer 模式可以正常的打印出数据，代价就是内存占用持续增长。我又以 个/MICROSECONDS 的速率来向下游发射事件，1分半钟的事件，内存占用疯涨到144mb。所以 Observable 是完全没有背压策略的，上游有多少数据照单全收，早晚会 OOM 的。

而使用 Flowable/Subscriber 模式在一开始就会报错的。抛出了 MissingBackpressureException 异常。仿佛在说：臣妾吃不下。
正确的使用方法如下：

```java
    Flowable.interval(1,TimeUnit.MILLISECONDS)
        .observeOn(Schedulers.computation())
        .subscribe(new Subscriber<Long>() {
          Subscription mSubscription = null;

          @Override
          public void onSubscribe(Subscription s) {
            mSubscription = s;
            mSubscription.request(1);
          }

          @Override
          public void onNext(Long aLong) {
            mSubscription.request(1);
            Log.i(ARIRUS, "onNext: "+aLong);
          }
          ...
        });
```

在下游调用 Subscription 的 request 方法，通知上游发送相应的数据，注意这样只是手动的将消耗事件和向上游请求事件保持一致，但是上游的生成事件的速度依然是不受影响的。

```java
    Flowable.interval(1,TimeUnit.MICROSECONDS)
        .observeOn(Schedulers.computation())
        .subscribe(new Subscriber<Long>() {
          Subscription mSubscription = null;

          @Override
          public void onSubscribe(Subscription s) {
            mSubscription = s;
            mSubscription.request(1);
          }

          @Override
          public void onNext(Long aLong) {
            mSubscription.request(1);
            Log.i(ARIRUS, "onNext: "+aLong);
          }
        ...
        });
```

依然会抛出：MissingBackpressureException: Can't deliver value 128 due to lack of requests 异常，因为就算我们是消费一个再请求一个，但是上游的速度实在是太快，跟不上。

因此如下的方法可能会更好一些，使用背压策略如果下游无法处理，抛弃掉相应的数据。

```java
    Observable.interval(1,TimeUnit.MICROSECONDS)
        .toFlowable(BackpressureStrategy.DROP)
        .observeOn(Schedulers.computation())
        .subscribe(new Subscriber<Long>() {
          Subscription mSubscription = null;

          @Override
          public void onSubscribe(Subscription s) {
            mSubscription = s;
            mSubscription.request(1);
          }

          @Override
          public void onNext(Long aLong) {
            mSubscription.request(1);
            Log.i(ARIRUS, "onNext: "+aLong);
          }
        ...
        });
07-18 15:58:17.855 I/MainActivity: onNext: 360
07-18 15:58:17.855 I/MainActivity: onNext: 361
07-18 15:58:17.855 I/MainActivity: onNext: 362
07-18 15:58:17.855 I/MainActivity: onNext: 363
07-18 15:58:17.855 I/MainActivity: onNext: 364
07-18 15:58:17.865 I/MainActivity: onNext: 436
07-18 15:58:17.865 I/MainActivity: onNext: 437
07-18 15:58:17.865 I/MainActivity: onNext: 438
07-18 15:58:17.865 I/MainActivity: onNext: 439
07-18 15:58:17.865 I/MainActivity: onNext: 440
```

对于 Subscriber 与 Observer 分别多出了两个方法：

    public void onSubscribe(Subscription s);
    public void onSubscribe(Disposable d);
Subscription 与 Disposable 定义如下：

```java
public interface Subscription {
    public void request(long n);
    public void cancel();
}

public interface Disposable {
    void dispose();
    boolean isDisposed();
}
```

这里的 Disposable 就是 1.X 版本的 Subscription ,用于取消订阅和判断订阅是否取消。而 Subscription 则是用于向上游请求数据。这里是简单的介绍下背压，详细内容我们放到下篇文章。

# Action Function

1.X 版本的 Action0 表示接受0个参数的，调用本身的接口，Action1，Action2 依次类推。新版本中，改为相应的 Action，Consumer，BiConsumer。
1.X 版本的 Func0 表示接受0个参数的，调用转换接口，Func2，Func3 依次类推。新版本中，改为相应的 Function，BiFunction，Function3。
这里太简单了，就不给例子了。

# 其他

操作符部分改的不少，不过大部分都是微调，这里我们调选几个变化比较大来介绍一下。

## subscribeWith 操作符

新加的操作符，用于返回订阅者

```java
public final <E extends Subscriber<? super T>> E subscribeWith(E subscriber) {
    subscribe(subscriber);
    return subscriber;
}
```

这是由于

```java
public final void subscribe(Subscriber<? super T> s)
```

返回为空，为了和 1.X 的 subscribe 返回的 Subscription 保持一致，因此新添加了这个操作符，

## compose 操作符

上一篇我们刚说过 compose 操作符，其参数 Transformer 继承于 Func1<Observable<T>, Observable<R>>。新版本中，改变不大：

```java
public final <R> Flowable<R> compose(FlowableTransformer<? super T, ? extends R> composer)

public interface FlowableTransformer<Upstream, Downstream> {
    Publisher<Downstream> apply(Flowable<Upstream> upstream);
}
```

## Processor

上文中，我们已经说过和 1.X 版本的 Subject 性质是一样的，可以将冷的激活成热的，二者的区别也就是在于 Processor 支持背压，但是 Subject 不支持。

# 小结

总体来说，你在 Rxjava 1 上习惯的操作完全适合在 Rxjava 2 上来使用，个人感觉如果项目中不考虑背压操作的话，使用 Rxjava 1 也是可以的。既然二者最大差距在于背压，那么我们下一篇着重来讲解一下背压的相关处理。再见。
