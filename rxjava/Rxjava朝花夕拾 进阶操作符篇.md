---
date : 2018-07-16T17:28:49+08:00
tags : ["Rxjava", "Android", "用法"]
title : "Rxjava朝花夕拾 进阶操作符篇"

---

本次源码分析是以 GitHub 上 RxJava 仓库 [1.3.0](https://github.com/ReactiveX/RxJava)版本来进行的分析，同时以[官方中文文档](https://legacy.gitbook.com/book/mcxiaoke/rxdocs/details)作为参考。

上篇中，我们分析对比了一些常用的操作符，但是只是直接的使用，没有分析其背后的原理。本篇，我们从两个操作符入手，分析下 Rxjava 的操作符源码。

<!--more-->

# Lift 与 Compose

Lift 和 Compose 属于两个比较复杂的操作符，并且不像之前的操作符，从名字就能看出其功能（直观），但此两个操作符重要性要比之前的操作符还要高，下面就来一点点分析。

- Lift  把源 Observable 按照转化成另外一个新的 Observable ，并对发射的数据进行操作。
- Compose 把源 Observable 按照转化成另外一个新的Observable， 针对源 Observable 来进行操作。

也就是说如果操作符是作用于源 Observable 发射出来的 item ，就用 Lift; 如果操作符是把源 Observable 当作一个整体来进行转换生成新的 Observable ，那么使用（尤其是使用现有的操作符来操作），就用 Compose。
我们先看下二者的区别：

```java
public <R> Observable<R> lift(final Operator<? extends R, ? super T> operator);
public <R> Observable<R> compose(Transformer<? super T, ? extends R> transformer);

public interface Operator<R, T> extends Func1<Subscriber<? super R>, Subscriber<? super T>>;
public interface Transformer<T, R> extends Func1<Observable<T>, Observable<R>>;
```

lift 的参数是 Operator ，其继承于 Func1<Subscriber, Subscriber> ，就是说 lift 是针对 Subscriber 进行操作，同理 compose 的参数 Transformer 继承于 Func1<Observable, Observable> ，说明 compose 是针对 Observable 来进行的操作。
这里先给出两个例子，看看两个操作符的应用：

```java
    Observable.just(1, 2)
        .lift((Observable.Operator<String, Integer>) subscriber -> new Subscriber<Integer>() {
          @Override
          public void onCompleted() {
            subscriber.onCompleted();
          }

          @Override
          public void onError(Throwable e) {
            subscriber.onError(e);
          }

          @Override
          public void onNext(Integer integer) {
            subscriber.onNext(String.valueOf(integer));
          }
        }).subscribe(string -> Log.i(ARIRUS, "lift: "+string +" "+ string.getClass()));
log:
I/ARIURS: lift: 1 class java.lang.String
I/ARIURS: lift: 2 class java.lang.String


    Observable.interval(1, TimeUnit.SECONDS)
        .compose(longObservable -> longObservable.take(10)
            .throttleLast(3, TimeUnit.SECONDS)
            .map(aLong -> aLong % 2 == 0))
        .subscribe(aBoolean -> Log.i(ARIRUS, "compose: " + aBoolean));

log:
I/ARIURS: compose: false
I/ARIURS: compose: true
I/ARIURS: compose: false
I/ARIURS: compose: false
```

上述代码分别是 lift 与 compose 的相关例子。lift 中，将 Integer 型元素转换为了 String 类型的元素，最后打印出来了元素的类型。compose 中，则是进行了一个过滤和转换操作，打印出最后的元素是否是偶数。

# Compose 源码分析

```java
public <R> Observable<R> compose(Transformer<? super T, ? extends R> transformer) {
    return ((Transformer<T, R>) transformer).call(this);
}
```

Compose 源码是非常简单的，只是调用了 Transformer 的 call 方法，因为我们之前说过，Transformer 其实是 Func1<Observable<T>, Observable<R>> ，因此返回的就是 Observable<R>。

# Lift 源码分析

```java
public final <R> Observable<R> lift(final Operator<? extends R, ? super T> operator) {
    return unsafeCreate(new OnSubscribeLift<T, R>(onSubscribe, operator));
}

public final class OnSubscribeLift<T, R> implements OnSubscribe<R> {

    final OnSubscribe<T> parent;

    final Operator<? extends R, ? super T> operator;

    public OnSubscribeLift(OnSubscribe<T> parent, Operator<? extends R, ? super T> operator) {
        this.parent = parent;
        this.operator = operator;
    }

    @Override
    public void call(Subscriber<? super R> o) {
        try {
            Subscriber<? super T> st = RxJavaHooks.onObservableLift(operator).call(o);
            try {
                // new Subscriber created and being subscribed with so 'onStart' it
                st.onStart();
                parent.call(st);
            } catch (Throwable e) {
                // localized capture of errors rather than it skipping all operators
                // and ending up in the try/catch of the subscribe method which then
                // prevents onErrorResumeNext and other similar approaches to error handling
                Exceptions.throwIfFatal(e);
                st.onError(e);
            }
        } catch (Throwable e) {
            Exceptions.throwIfFatal(e);
            // if the lift function failed all we can do is pass the error to the final Subscriber
            // as we don't have the operator available to us
            o.onError(e);
        }
    }
}
```

Lift 其实就是调用了 unsafeCreate 方法，返回了一个新的 Observable，和我们通常的写的 Observable.create 方法没啥区别。其参数是 OnSubscribeLift 实例，实现了 OnSubscribe 接口。其构造函数将上游 OnSubscribe 和 具体针对每个 item 的操作传入了进来。继续看其 call 方法，我们知道 call
 方法是 OnSubscribe 的核心，当 call 方法调用时，传入进来了订阅者，调用 operator 的 call 方法，将 Subscriber 进行转换，将新的 Subscriber 传送给 parent，让他来调用新的 Subscriber 的相关方法。所以这里有两个地方需要注意：

    OnSubscribeLift 在构造的时候，传入的是上游的 OnSubscribe 而非 Observable。
    OnSubscribeLift 的 call 方法在调用的时候，将 Operator 操作完后的 Subscriber 传给了 parent。

这两部分就是 lift 操作符的核心，和别的操作符不一样的地方。

## Filter 与 Lift 对比

上述描述可能比较晦涩，这里我们根据这两个操作符的用法来进行下对比。

```java
Observable.create((Observable.OnSubscribe<Integer>) subscriber -> {
  subscriber.onNext(1);
  subscriber.onNext(2);
})
    .filter(integer -> integer % 2 == 0)
    .lift((Observable.Operator<String, Integer>) subscriber -> new Subscriber<Integer>() {
      @Override
      public void onCompleted() {
        subscriber.onCompleted();
      }

      @Override
      public void onError(Throwable e) {
        subscriber.onError(e);
      }

      @Override
      public void onNext(Integer integer) {
        subscriber.onNext(String.valueOf(integer));
      }
    })
    .subscribe(string -> Log.i(ARIRUS, "call: " + string + " " + string.getClass()));
```

这段代码的意思是：上游发射两个 Integer 数据，经过 Filter 操作符，只有偶数才能到下游，经过 Lift 操作符，将 Integer 类型数据转换成 String 类型数据后，继续向下游发射过去。

我们来看其流程：

1. Observable.create 创建 Observable_1（值为 Observable@4446） 包含 onSubscribe_1 （值为 Activity$lambda@4429）

```java
public static <T> Observable<T> create(OnSubscribe<T> f) {
    return new Observable<T>(RxJavaHooks.onCreate(f));
}
```

2. filter 调用 unsafeCreate 生成 Observable_2（值为 Observable@4540）, 同时将 Observable_1 作为参数传入 OnSubscribeFilter 实例，包含 onSubscribe_2（值为 OnSubscribeFilter@4535）

```java
public final Observable<T> filter(Func1<? super T, Boolean> predicate) {
    return unsafeCreate(new OnSubscribeFilter<T>(this, predicate));
}
```

3. lift 调用 unsafeCreate 生成 Observable_3 （值为 Observable@4574）, 同时将 onSubscribe_2 作为参数传入 OnSubscribeLift 实例，包含 onSubscribe_3（值为 OnSubscribeLift@4575）

```java
public final <R> Observable<R> lift(final Operator<? extends R, ? super T> operator) {
    return unsafeCreate(new OnSubscribeLift<T, R>(onSubscribe, operator));
}
```

4. subscribe 中调用了 onSubscribe_3，让其调用最后的 subscriber。因为 onSubscribe_3 是一个 OnSubscribeLift 实例，因此调用了其 call 方法，回到了 OnSubscribeLift 类内部。

```java
static <T> Subscription subscribe(Subscriber<? super T> subscriber, Observable<T> observable) {
....
RxJavaHooks.onObservableStart(observable, observable.onSubscribe).call(subscriber); //实质就是调用的 observable.onSubscribe.call(subscriber);
....
}


public static <T> Observable.OnSubscribe<T> onObservableStart(Observable<T> instance, Observable.OnSubscribe<T> onSubscribe) {
    Func2<Observable, Observable.OnSubscribe, Observable.OnSubscribe> f = onObservableStart;
    if (f != null) {
        return f.call(instance, onSubscribe);
    }
    return onSubscribe;
}
```

5. 调用了 OnSubscribeLift 内部的 operator 的 call 方法，返回了一个 Subscriber<Integer>，将得到的 Subscriber 传递给 parent，让其调用 call 方法。

```java
public final class OnSubscribeLift<T, R> implements OnSubscribe<R> {

    public OnSubscribeLift(OnSubscribe<T> parent, Operator<? extends R, ? super T> operator) {
        this.parent = parent;
        this.operator = operator;
    }

    @Override
    public void call(Subscriber<? super R> o) {
        ...
        Subscriber<? super T> st = RxJavaHooks.onObservableLift(operator).call(o); //根据 Operator 生成新的 Subscriber
        parent.call(st); // 直接由上游的 OnSubscribe 接管 Subscriber
        ...
    }
}
```

6. parent 就是上面的作为参数传递过来的 onSubscribe_2 ，因此就是调用 OnSubscribeFilter 的 call 方法。在 call 方法内部，将下游的 Subscriber 与 filter 的过滤操作封装成一个新的 Subscriber ，同时交给上游的 Observable 进行订阅。

```java
public final class OnSubscribeFilter<T> implements OnSubscribe<T> {
    ...
    @Override
    public void call(final Subscriber<? super T> child) {
        FilterSubscriber<T> parent = new FilterSubscriber<T>(child, predicate);
        child.add(parent);
        source.unsafeSubscribe(parent);
    }
    ...
}
```

7. 上游 Observable 就是 Observable_1，其 onSubscribe_1 调用下游传入的 subscriber，则可以把需要发射的数据发射出去。

```java
Observable.create((Observable.OnSubscribe<Integer>) subscriber -> {
  subscriber.onNext(1);
  subscriber.onNext(2);
})
...
```

这样其实是有两条线：

    1. Observable 按照从上游到下游的顺序依次建立，可能保存上一个 Observable 或者 OnSubscribe。
    2. Subscriber 按照从下游到上游的顺序依次建立，每一个 Subscriber 都是由之前的 Subscriber 构造而来。

# 后记

本篇主要分析 Lift 与 Compose 两个操作符，并结合 Filter 对比了与 Lift 的异同，分析了整个流从建立到订阅的流程。下一篇中，我们结合 Rxjava 2.X 对比 1.X 看看都有哪些部分的改进。