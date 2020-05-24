---
date : 2018-05-02T21:29:20+08:00
tags : ["Rxjava", "Android", "用法"]
title : "Rxjava朝花夕拾 概念篇"

---

本次源码分析是以 GitHub 上 RxJava 仓库 [1.3.0](https://github.com/ReactiveX/RxJava)版本来进行的分析，同时以[官方中文文档](https://legacy.gitbook.com/book/mcxiaoke/rxdocs/details)作为参考。

# 为什么要写这个系列

Rxjava 在开发中的重要性已无须多言，可以说现在每新建一个 Android 工程，首先就是要 gradle 中配置 Rxjava 的依赖。本系列不会是一个教科书性质的东西，太过复杂也没有必要都8012年，还有不会用Rxjava的开发？更多会写个总结性质的系列，包括用法，思路，例子等等，当然也有一些查漏补缺，相似对比之类。好了开始吧。

<!--more-->

# 概览

响应式编程在文档上，给的定义为 Rx = Observables + LINQ + Schedulers，翻译来看就是：观察者模式、流式编程、线程切换的三者有机结合。概括的很清晰，详细来说就是：

    观察者（Observer） 和 被观察者（Observable）
    流式操作符：just from map flatmap filter observeOn
    线程切换：IO线程 Computation线程 主线程
那么概念篇也就是按照这个顺序来进行介绍。

# Observable

Observable 可能是最熟悉的 Rxjava 组件，其意义是被观察者，也就是其 emit item 发送给观察者，在 1.X 版本中,我们会使用 create 创建符或者 just from 等来生成一个 Observable 。通常我们使用操作符生成的 Observable 是 cold 的，所谓 cold Observable 就是 Observable 会一直等待，直到有观
察者订阅它才开始发射数据，因此这个观察者可以确保会收到整个数据序列。

```java
    Observable<Long> observable1 = Observable.create(new Observable.OnSubscribe<Long>() {
      @Override
      public void call(Subscriber<? super Long> subscriber) {
        Observable.interval(10, TimeUnit.MILLISECONDS)
            .take(10)
            .subscribe(aLong -> subscriber.onNext(aLong));
      }
    });
    observable1.subscribe(integer -> Log.i(ARIRUS, "xxx3: " + integer));

    try {
      Thread.sleep(50L);
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
    observable1.subscribe(integer -> Log.i(ARIRUS, "xxx4: " + integer));


log:
    I/ARIURS: xxx3: 0
    I/ARIURS: xxx3: 1
    I/ARIURS: xxx3: 2
    I/ARIURS: xxx3: 3
    I/ARIURS: xxx3: 4
    I/ARIURS: xxx3: 5
    I/ARIURS: xxx4: 0
    I/ARIURS: xxx3: 6
    I/ARIURS: xxx4: 1
    I/ARIURS: xxx3: 7
    I/ARIURS: xxx4: 2
    I/ARIURS: xxx3: 8
    I/ARIURS: xxx4: 3
    I/ARIURS: xxx3: 9
    I/ARIURS: xxx4: 4
    I/ARIURS: xxx4: 5
    I/ARIURS: xxx4: 6
    I/ARIURS: xxx4: 7
    I/ARIURS: xxx4: 8
```

上述例子中，两次订阅,得到了两个独立的响应序列。但是，对于某些环境 cold Observable 并不适用，例如网络环境的变化，当订阅时，我们不会想知道“历史上”所有的网络的变化，我只想知道上一次的变化，因此 hot Observable 是有必要的，一个"热"的
Observable 可能一创建完就开始发射数据，因此所有后续订阅它的观察者可能从序列中间的
某个位置开始接受数据（有一些数据错过了）。

RxJava 中使用 ConnectableObservable 来表示一个 hot Observable,可连接的 Observable 在
被订阅时并不开始发射数据，只有在它的 connect() 被调用时才开始。用这个方法，你可
以等待所有的观察者都订阅了 Observable 之后再开始发射数据。而使用 publish() 方法，可以将一个 cold Observable 转换成 hot Observable

```java
    //
    Observable<Long> observable1 = Observable.create(new Observable.OnSubscribe<Long>() {
      @Override
      public void call(Subscriber<? super Long> subscriber) {
        Observable.interval(10, TimeUnit.MILLISECONDS)
            .take(10)
            .subscribe(aLong -> subscriber.onNext(aLong));
      }
    }).publish().autoConnect(); //此处是不同点 转换为一个 ConnectableObservable 并开始 emit item
    observable1.subscribe(integer -> Log.i(ARIRUS, "xxx3: " + integer));

    try {
      Thread.sleep(50L);
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
    observable1.subscribe(integer -> Log.i(ARIRUS, "xxx4: " + integer));

log:
    I/ARIURS: xxx3: 0
    I/ARIURS: xxx3: 1
    I/ARIURS: xxx3: 2
    I/ARIURS: xxx3: 3
    I/ARIURS: xxx3: 4
    I/ARIURS: xxx3: 5
    I/ARIURS: xxx4: 5
    I/ARIURS: xxx3: 6
    I/ARIURS: xxx4: 6
    I/ARIURS: xxx3: 7
    I/ARIURS: xxx4: 7
    I/ARIURS: xxx3: 8
    I/ARIURS: xxx4: 8
    I/ARIURS: xxx3: 9
    I/ARIURS: xxx4: 9
```

“Subscriber2” 从第5个数据开始接受，之前的全部丢弃。

# Single 与 Completable

Single 平时可能用的比较少，简单介绍下。Single类似于Observable，不同的是，它总是只发射一个值，或者一个错误通知，而不是发射一系列的值。可以用于倒计时，网络请求这种，只发射一个值的情况。订阅Single只需要两个方法：

- onSuccess - Single发射单个的值到这个方法
- onError - 如果无法发射需要的值，Single发射一个Throwable对象到这个方法

```java
Single.just(6).subscribe(integer -> Log.i(ARIRUS, "onCreate: "+integer));
```

Completable 也类似 Observable，只是反射结束通知，或者异常：

- onComplete - 订阅完成后发射
- onError - 发射了一个异常

# Subject

Subject 同时充当了 Observer 和 Observable 的角色。由于一个Subject订阅一个Observable，它可以触发这个Observable开始发射数据（如果那个
Observable是"冷"的--就是说，它等待有订阅才开始发射数据）。因此有这样的效果，Subject
可以把原来那个"冷"的Observable变成"热"的。

```java
    Observable<Long> observable = Observable.create(new Observable.OnSubscribe<Long>() {
      @Override
      public void call(Subscriber<? super Long> subscriber) {
        Observable.interval(10, TimeUnit.MILLISECONDS, Schedulers.computation())
            .take(10)
            .subscribe(subscriber::onNext);
      }
    }).observeOn(Schedulers.newThread());

    PublishSubject<Long> subject = PublishSubject.create();
    observable.subscribe(subject);

    subject.subscribe(aLong -> Log.i(ARIRUS, "subject subscribe1: "+aLong));

    try {
      Thread.sleep(20L);
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
    subject.subscribe(aLong -> Log.i(ARIRUS, "subject subscribe2: "+aLong));

```

这里我们并没将一个 Observable 转换为 ConnectableObservable ，而是使用 Subject 对象.当 Subject 作为 Subscriber 时，它可以订阅目标 Cold Observable 使对方开始发送事件。同时它又作为Observable 转发或者发送新的事件，让 Cold Observable 借助 Subject 转换为 Hot Observable。這就是文檔上所谓的把“冷的”变成“热的”。

Subject按功能分有4类：

- AsyncSubject 一个AsyncSubject只在原始Observable完成后，发射来自原始Observable的最后一个值。它会把这最后一个值发射给任何后续的观察者。
- BehaviorSubject 当观察者订阅BehaviorSubject时，它开始发射原始Observable最近发射的数据（如果此时还没有收到任何数据，它会发射一个默认值），然后继续发射其它任何来自原始Observable的数据。
- PublishSubject PublishSubject只会把在订阅发生的时间点之后来自原始Observable的数据发射给观察者。
- ReplaySubject ReplaySubject会发射所有来自原始Observable的数据给观察者，无论它们是何时订阅的。

四种例子如下：

```java
    Action1<String> subs1 = s -> Log.i(ARIRUS, "subs1: "+s);
    Action1<String> subs2 = s -> Log.i(ARIRUS, "subs2: "+s);

    //AsyncSubject
    AsyncSubject<String> subject = AsyncSubject.create();
    subject.subscribe(subs1);
    subject.onNext("A");
    subject.onNext("B");
    subject.subscribe(subs2);
    subject.onNext("C");
    subject.onNext("D");
    subject.onCompleted();

    //log
    I/ARIRUS: subs1: D
    I/ARIRUS: subs2: D //不管什么时候订阅，只发射最后一个值

    //BehaviorSubject
    BehaviorSubject<String> subject = BehaviorSubject.create();
    subject.subscribe(subs1);
    subject.onNext("A");
    subject.onNext("B");
    subject.subscribe(subs2);
    subject.onNext("C");
    subject.onNext("D");
    subject.onCompleted();

    //log
    I/ARIRUS: subs1: A
    I/ARIRUS: subs1: B
    I/ARIRUS: subs2: B
    I/ARIRUS: subs1: C
    I/ARIRUS: subs2: C
    I/ARIRUS: subs1: D
    I/ARIRUS: subs2: D//不管什么时候订阅，发射离自己最近的一个值

    //PublishSubject
    PublishSubject<String> subject = PublishSubject.create();
    subject.subscribe(subs1);
    subject.onNext("A");
    subject.onNext("B");
    subject.subscribe(subs2);
    subject.onNext("C");
    subject.onNext("D");
    subject.onCompleted();

    //log
    I/ARIRUS: subs1: A
    I/ARIRUS: subs1: B
    I/ARIRUS: subs1: C
    I/ARIRUS: subs2: C
    I/ARIRUS: subs1: D
    I/ARIRUS: subs2: D //发射订阅之后的值

    //ReplaySubject 发射所有的值
    ReplaySubject<String> subject = ReplaySubject.create();
    subject.subscribe(subs1);
    subject.onNext("A");
    subject.onNext("B");
    subject.subscribe(subs2);
    subject.onNext("C");
    subject.onNext("D");
    subject.onCompleted();

    //log
    I/ARIRUS: subs1: A
    I/ARIRUS: subs1: B
    I/ARIRUS: subs2: A
    I/ARIRUS: subs2: B
    I/ARIRUS: subs1: C
    I/ARIRUS: subs2: C
    I/ARIRUS: subs1: D
    I/ARIRUS: subs2: D //无论是什么时候订阅，发射所有的值
```

Subject 用处不同于上述 Observable create 生成的被观察者。Subject 解决的需求就是不确定什么时候可以发出事件。假设我们有一些数据希望通过 RxJava 来发出，但我们并不确定这些数据什么时候到来、以及有多少数据。这样的情况，是用 Subject 最好的例子，如 RxBus，RecyclerView 中 item 的点击事件，使用 Subject 来发布、订阅是最合适的。

# 操作符

因为这部分内容较多因此放到后续一篇来单独写，这里就摸了。

# 小结

这篇里面，我们主要介绍了响应式编程相关概念，着重介绍了不同类型的 Subject 对于初学来说，可能不好理解，如果有一定的相关编程经验，就会知道这些东西是非常重要的。好了，下篇我们就开始 Rxjava 的操作符的相关介绍。再见。
