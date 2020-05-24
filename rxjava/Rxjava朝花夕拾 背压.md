---
date : 2018-07-19T14:36:49+08:00
tags : ["Rxjava", "Android", "用法"]
title : "Rxjava朝花夕拾 背压篇"

---

本次源码分析是以 GitHub 上 RxJava 仓库 [2.1.1](https://github.com/ReactiveX/RxJava)版本来进行的分析，同时以[官方中文文档](https://legacy.gitbook.com/book/mcxiaoke/rxdocs/details)作为参考。

上一篇文章，我们对比了一下 Rxjava 1.X 与 2.X 版本的差异点，简单来说就是，对于背压（Backpressure）有了更好的支持，同时也简单的介绍了背压，本篇我们再来进行下详细的介绍背压和背压策略。本篇参考了颇多[Android RxJava：一文带你全面了解 背压策略](https://blog.csdn.net/carson_ho/article/details/79081407) 在此感谢作者。

<!--more-->

# 背压

先说下基本定义，所谓背压就是：上游的事件发布者的事件发布速率，大于，下游的事件消费者的事件消费速率。
这个定义很好理解，就像是给油壶倒油，如果倒油的流量大于漏斗的上面那个“缓冲装置”，那么油就会流到地上来，因此从本质上来说就是一个“缓冲区的溢出问题”。那么在理解这种情况下，我们再分别同步订阅和异步订阅的背压区别。

## 同步订阅

所谓同步订阅就是订阅和观察都在同一个线程，不适用线程切换或者仅调用 subscribeOn 方法都可以达到目的，以下例子使用前者。

### 下游通知

```java
Flowable.interval(1, TimeUnit.MICROSECONDS).subscribe(integer -> {
  try {
    Thread.sleep(1000);
  } catch (Exception e) {
    e.printStackTrace();
  }
  Log.i(ARIRUS, "onCreate: " + integer);
});
```

这是一个很简单的例子，生产者调用 interval 来产生数据，速率是每微秒一个数据，而在观察者是每秒才能消耗一个数据。按说日志依然很正常并没有抛出 MissingBackPresureException ，所以这里是没有产生异常的。
我们换个写法：

```java
Flowable.interval(1, TimeUnit.MICROSECONDS).subscribe(new Subscriber<Long>() {
  @Override
  public void onSubscribe(Subscription s) {

  }

  @Override
  public void onNext(Long aLong) {
    try {
      Thread.sleep(1000);
    } catch (Exception e) {
      e.printStackTrace();
    }
    Log.i(ARIRUS, "onCreate: " + aLong);
  }

  @Override
  public void onError(Throwable t) {
    t.printStackTrace();
  }
...
});

Log:
io.reactivex.exceptions.MissingBackpressureException: Can't deliver value 0 due to lack of requests
```

直接就抛异常（缺少 request ,无法传递数据0）。如果按照上篇文章一样修改在 onSubscribe 和 onNext 分别调用 Subscription 的 request 方法，则不会抛出异常。因为 Rxjava 的背压处理是基于动态拉取的，即观察者发送 request 的个数给被观察者，被观察者发送出相应个数的数据给下游，这样以此类推。因此对于同步订阅也是会抛出 MissingBackpressureException 异常，这里原因其实也是上下游对于数据的处理速率不匹配造成的，上游发送4个，下游只接受3个，因此最后一个数据在下游的处理速率接近于无穷，当然是不匹配的。
之前的写法没产生背压的原因很简单，因为 Rxjava 通过封装，他替你做了：

```java
public final Disposable subscribe(Consumer<? super T> onNext) {
    return subscribe(onNext, Functions.ON_ERROR_MISSING,
            Functions.EMPTY_ACTION, FlowableInternalHelper.RequestMax.INSTANCE);
}

public final Disposable subscribe(Consumer<? super T> onNext, Consumer<? super Throwable> onError,
        Action onComplete, Consumer<? super Subscription> onSubscribe) {
    ...
    LambdaSubscriber<T> ls = new LambdaSubscriber<T>(onNext, onError, onComplete, onSubscribe);

    subscribe(ls);

    return ls;
}

public final class LambdaSubscriber<T>{
    ...
    @Override
    public void onSubscribe(Subscription s) {
        if (SubscriptionHelper.setOnce(this, s)) {
            try {
                onSubscribe.accept(this);
            } catch (Throwable ex) {
                Exceptions.throwIfFatal(ex);
                s.cancel();
                onError(ex);
            }
        }
    }
    ...
}

public enum RequestMax implements Consumer<Subscription> {
    INSTANCE;
    @Override
    public void accept(Subscription t) throws Exception {
        t.request(Long.MAX_VALUE);
    }
}
```

根据传入的 Consumer 重新封装一个 LambdaSubscriber，而其 onSubscribe 时调用的就是 RequestMax.INSTANCE 的 accept 方法，直接 request(Long.MAX_VALUE)。相当于告诉上游来者不拒，因此不会抛出异常。

### 上游反馈

现在我们来修改一下上游的事件生成方式：

```java
Flowable.create(new FlowableOnSubscribe<Long>() {
  @Override
  public void subscribe(FlowableEmitter<Long> emitter) throws Exception {
    long i = 0;
    while (emitter.requested()!=0) {
      Log.i(ARIRUS, "subscribe: "+i+" "+emitter.requested());
      emitter.onNext(i++);
    }
  }
}, BackpressureStrategy.ERROR).subscribe(new Subscriber<Long>() {
  @Override
  public void onSubscribe(Subscription s) {
    s.request(100);
  }

  @Override
  public void onNext(Long aLong) {
    try {
      Thread.sleep(5000);
    } catch (Exception e) {
      e.printStackTrace();
    }
    Log.i(ARIRUS, "onCreate: " + aLong);
  }
...
});

Log:
07-30 16:20:39.588 I/ARIRUS: subscribe: 0 100
07-30 16:20:44.593 I/ARIRUS: onCreate: 0
07-30 16:20:44.593 I/ARIRUS: subscribe: 1 99
07-30 16:20:49.588 I/ARIRUS: onCreate: 1
07-30 16:20:49.588 I/ARIRUS: subscribe: 2 98
07-30 16:20:54.582 I/ARIRUS: onCreate: 2
07-30 16:20:54.582 I/ARIRUS: subscribe: 3 97
07-30 16:20:59.587 I/ARIRUS: onCreate: 3
```

这次我们在上游，一个死循环中来产生数据发射给下游。因为在下游订阅时，告诉上游只能处理100个数据，因此 emitter.requested() 是从100逐次递减到0。同时在下游处理数据时，上游的发射操作被堵塞。
因此，在同步订阅的情况下，上下游之间没有缓存区域，下游阻塞上游也会阻塞，因此在上下游处理同样数量的情况下，上游的产生速率和下游的消耗速率是相等的。这也是为什么同步订阅基本不需要考虑背压。

## 异步订阅

不同于同步订阅，异步订阅有相应的线程切换，因此上下游之间有一个缓冲区。导致了且背压与同步订阅完全不同。

```java
Flowable.create(new FlowableOnSubscribe<Long>() {
  @Override
  public void subscribe(FlowableEmitter<Long> emitter) throws Exception {
    long i = 0;
    while (emitter.requested()!=0) {
      Log.i(ARIRUS, "subscribe: "+i+" "+emitter.requested());
      emitter.onNext(i++);
    }
  }
}, BackpressureStrategy.ERROR)
    .subscribeOn(Schedulers.newThread())
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(new Subscriber<Long>() {
  @Override
  public void onSubscribe(Subscription s) {
    s.request(100);
  }

  @Override
  public void onNext(Long aLong) {
    try {
      Thread.sleep(5000);
    } catch (Exception e) {
      e.printStackTrace();
    }
    Log.i(ARIRUS, "onCreate: " + aLong);
  }
...
});

log
07-30 19:16:06.660 I/ARIRUS: subscribe: 0 128
07-30 19:16:06.661 I/ARIRUS: subscribe: 1 127
07-30 19:16:06.661 I/ARIRUS: subscribe: 2 126
07-30 19:16:06.661 I/ARIRUS: subscribe: 3 125
```

这里我们看到，代码只是增加了切换线程的操作，除此之外没有别的操作。但是上游 emitter.requested() 为128，而非我们设置的100。其实这128就是中间的缓冲区的大小，即所谓的“油壶漏斗”的大小，因为订阅和观察不在同一个线程中，因此一个缓冲区是十分必要的。因此如果我们在缓冲区达到128之后，依然向其传输数据就会产生 MissingBackpressureException 。例如这里我们将 FlowableOnSubscribe 的 subscribe 改写为：

```java
public void subscribe(FlowableEmitter<Long> emitter) throws Exception {
    long i = 0;
    while (i<200) {
      Log.i(ARIRUS, "subscribe: "+i+" "+emitter.requested());
      emitter.onNext(i++);
    }
}

log:
07-30 19:32:24.529 I/ARIRUS: subscribe: 126 2
07-30 19:32:24.529 I/ARIRUS: subscribe: 127 1
07-30 19:32:24.530 I/ARIRUS: subscribe: 128 0
07-30 19:32:24.530 I/ARIRUS: subscribe: 129 0

MissingBackpressureException: create: could not emit value due to lack of requests
```

那这里其实的意思就是：“无视”掉下游的request，就是无视掉我们这里设置的“100”的请求。直接根据上游128个缓存数据来进行传递，而且最多只能传递128个，超过的话会抛异常。
但其实并不是真的无视，我们来看下下面的请求：

```java
//上游订阅
public void subscribe(FlowableEmitter<Long> emitter) throws Exception {
    long i = 0;
    while (true)
    while (emitter.requested()!=0 && i<300) {
      Log.i(ARIRUS, "subscribe: "+i+" "+emitter.requested());
      emitter.onNext(i++);
    }
}
//下游请求
public void onSubscribe(Subscription s) {
    s.request(223);
}
log：
07-30 19:44:48.419 I/ARIRUS: subscribe: 0 128
07-30 19:44:48.419 I/ARIRUS: subscribe: 1 127
...
07-30 19:44:48.425 I/ARIRUS: subscribe: 127 1
07-30 19:44:48.548 I/ARIRUS: onCreate: 0  
07-30 19:44:48.648 I/ARIRUS: onCreate: 1
...
07-30 19:44:58.094 I/ARIRUS: onCreate: 95
07-30 19:44:58.094 I/ARIRUS: subscribe: 128 96
07-30 19:44:58.094 I/ARIRUS: subscribe: 129 95
...
07-30 19:44:58.098 I/ARIRUS: subscribe: 223 1
07-30 19:44:58.195 I/ARIRUS: onCreate: 96
07-30 19:44:58.295 I/ARIRUS: onCreate: 97
...
07-30 19:45:07.745 I/ARIRUS: onCreate: 191
07-30 19:45:07.745 I/ARIRUS: subscribe: 224 96
07-30 19:45:07.745 I/ARIRUS: subscribe: 225 95
...
07-30 19:45:07.749 I/ARIRUS: subscribe: 299 21
07-30 19:45:07.846 I/ARIRUS: onCreate: 192
07-30 19:45:07.946 I/ARIRUS: onCreate: 193
...
07-30 19:45:18.602 I/ARIRUS: onCreate: 222
```

这里只发射300个数据，并且在 emitter.requested() 为0时，停止发射数据，直到 emitter.requested() 大于0时才会又开始。直到发射了300个数据，完全结束发射，同时下游只是请求223个数据。
我们看其发射过程是这样的：上游发射128个数据，下游处理96个数据，上游发射96个数据，下游处理96个数据，上游发射76个数据，下游处理31个数据。上游总共发射了300个数据，下游总共消费了223个数据。
因此对于异步订阅我们可以进行以下总结：

    1.缓冲区默认为128，如果上游 emitter.requested 为0时，继续发射数据会造成 MissingBackpressureException。
    2.在下游消费96个数据之后上游才会继续发射数据，否则上游不会产生新的数据。
    3.上游产生数据的数量和下游消费数据的数量是相互独立的。当然这是建立在上游产生数据比下游消费数据多大的前提下，反之上游有多少数据下游就会消费多少。

# 背压策略

通过上面我们大致了解了，在同步或者异步的情况下背压不同的情况。简而言之一句话，就是上游的事件大于下游的事件，因此当这种情况出现时，使用背压策略来避免出现抛出背压异常。
Flowable的几种背压策略：

    1. BackpressureStrategy.MISSING：OnNext事件没有任何缓存和丢弃，下游要处理任何溢出。
    2. BackpressureStrategy.ERROR：缓存区默人大小128，流速不均衡时发射MissingBackpressureException信号。
    3. BackpressureStrategy.BUFFER：缓存区不限制大小，使用不当仍会OOM。
    4. BackpressureStrategy.DROP：缓存最近的nNext事件。
    5. BackpressureStrategy.LATEST：缓存区会保留最后的OnNext事件，覆盖之前缓存的OnNext事件。

`MISSING` 这种情况下，上游忽略下游的 request 数量，所有的事件都直接传给下游，对于同步订阅的情况下，不会抛出异常，对于异步订阅会提示缓存区满了。
`ERROR` 这种策略是很多创建操作符的默认背压策略，如 interval ,这个策略默认就是如果上游发射的数据数量超过下游可以接受的数量就抛出异常。
`BUFFER` 这个策略使得 Flowable 的行为像 Observable一样，即有个无限大的 buffer 区，就算是爆内存也不会抛出背压异常。
`DROP` 与 `LASTET` 都是最常用的背压策略，如果上游在发射数据时下游无法接收数据则将多余的数据给丢弃掉，`LASTET` 会将最后一个数据保存起来发射给下游。
我们来看以下两个例子：

```java
Flowable
    .create(new FlowableOnSubscribe<Long>() {
      @Override
      public void subscribe(FlowableEmitter<Long> emitter) throws Exception {
        long i = 0;
        while (i < 2000) {
          emitter.onNext(i++);
          Log.i(ARIRUS, "emit: " + i);
          Thread.sleep(10);
        }
      }
    }, BackpressureStrategy.DROP)
    .subscribeOn(Schedulers.newThread())
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(new Subscriber<Long>() {
      @Override
      public void onSubscribe(Subscription s) {
        s.request(Long.MAX_VALUE);
      }

      @Override
      public void onNext(Long aLong) {
        try {
          Thread.sleep(100);
        } catch (Exception e) {
          e.printStackTrace();
        }
        Log.i(ARIRUS, "onCreate: " + aLong);
      }
        ...
    });

log:
08-03 10:44:17.836 I/ARIRUS: onCreate: 0
08-03 10:44:17.936 I/ARIRUS: onCreate: 1
...
08-03 10:44:30.688 I/ARIRUS: onCreate: 127
08-03 10:44:30.798 I/ARIRUS: onCreate: 885
08-03 10:44:30.898 I/ARIRUS: onCreate: 886
...
08-03 10:44:40.448 I/ARIRUS: onCreate: 980
08-03 10:44:40.548 I/ARIRUS: onCreate: 1723
08-03 10:44:40.648 I/ARIRUS: onCreate: 1724
...
08-03 10:44:50.107 I/ARIRUS: onCreate: 1817
08-03 10:44:50.207 I/ARIRUS: onCreate: 1818
```

最开始的128个数据发射完成后，直接从885开始发射，之间的数据均已丢失，在发射了96个数据，即980发射完成后，重复上一次动作。因为发射到1818时，上游的2000个数据早已发射完成，因此下游也没有需要接受的数据，因此1818就是最后一个数据。

```java
Flowable
    .create(new FlowableOnSubscribe<Long>() {
      @Override
      public void subscribe(FlowableEmitter<Long> emitter) throws Exception {
        long i = 0;
        while (i < 2000) {
          emitter.onNext(i++);
          Log.i(ARIRUS, "emit: " + i);
          Thread.sleep(10);
        }
      }
    }, BackpressureStrategy.LASTET)
    .subscribeOn(Schedulers.newThread())
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(new Subscriber<Long>() {
      @Override
      public void onSubscribe(Subscription s) {
        s.request(Long.MAX_VALUE);
      }

      @Override
      public void onNext(Long aLong) {
        try {
          Thread.sleep(100);
        } catch (Exception e) {
          e.printStackTrace();
        }
        Log.i(ARIRUS, "onCreate: " + aLong);
      }
        ...
    });

log:
08-03 11:03:24.885 I/ARIRUS: onCreate: 0
08-03 11:03:24.985 I/ARIRUS: onCreate: 1
...
08-03 11:03:37.777 I/ARIRUS: onCreate: 127
08-03 11:03:37.877 I/ARIRUS: onCreate: 871
08-03 11:03:37.977 I/ARIRUS: onCreate: 872
...
08-03 11:03:47.497 I/ARIRUS: onCreate: 966
08-03 11:03:47.597 I/ARIRUS: onCreate: 1734
08-03 11:03:47.697 I/ARIRUS: onCreate: 1735
...
08-03 11:03:57.186 I/ARIRUS: onCreate: 1828
08-03 11:03:57.286 I/ARIRUS: onCreate: 1829
08-03 11:03:57.396 I/ARIRUS: onCreate: 1999
```

'LASTET' 前面发射与 'DROP' 相同，不同点在于最后一段。最后一段的96个数据完成后，还有一个1999单独发射了出来。

# 小结

本篇中，我们通过各种例子了解背压在同步订阅与异步订阅的异同，并知道如何控制上游发射速度来达到上下游的数据处理速度平衡，避免背压的产生。最后还分析了几个背压策略在上下游速率不平衡的情况下，通过抛弃多余事件使得上下游平衡，避免背压的产生。