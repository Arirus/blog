---
date : 2018-07-12T15:06:21+08:00
tags : ["Rxjava", "Android", "用法"]
title : "Rxjava朝花夕拾 基本操作符篇"

---

本次源码分析是以 GitHub 上 RxJava 仓库 [1.3.0](https://github.com/ReactiveX/RxJava)版本来进行的分析，同时以[官方中文文档](https://legacy.gitbook.com/book/mcxiaoke/rxdocs/details)作为参考。

上一篇中，主要写了两个重要的概念：Observable, Subject。 (Observer Subscriber 不做描写) 本篇中，我们着重来对比几个常用的操作符，并结合具体的环境给出例子。由于大部分的操作符都有多个参数类型，这里根据最基本的参数类型，解释对比其原理。

<!--more-->

# 创建操作符

## Empty Just(null)

- Empty 创建一个不发射任何数据但是正常终止的 Observable.
- Just(null) 返回一个发射 null 值的Observable.不是什么数据都不会发射

## Timer Interval Delay 

- Timer 创建一个Observable，它在一个给定的延迟后发射一个特殊的值。
- Interval 创建一个按固定时间间隔发射整数序列的Observable。
- Delay 创建一个按固定时间间隔发射整数序列的Observable。

严格来说 Delay 不应该放在这里来说，毕竟不是创作操作符，不过由于在效果上类似 timer 这里就一并说了。

```java
Observable.timer(3,TimeUnit.SECONDS).subscribe(aLong -> Log.i(ARIRUS, "timer subscribe"));

Observable.just(1).delay(3,TimeUnit.SECONDS).subscribe(aLong -> Log.i(ARIRUS, "delay subscribe"));

Observable.interval(3,TimeUnit.SECONDS).subscribe(aLong -> Log.i(ARIRUS, "interval subscribe"));

log:
16:28:12.022  I/ARIURS: timer subscribe
16:28:12.032  I/ARIURS: delay subscribe
16:28:12.032  I/ARIURS: interval subscribe
16:28:15.035  I/ARIURS: interval subscribe
16:28:18.038  I/ARIURS: interval subscribe
```

可以看到，timer 是延迟发射一个0值，而 delay 是将流中的某一个 item 延长发送到下游，二者最大的区别就在这里。interval 则是周期性的发送 item 从0开始。我们可以使用 interval 来写一个倒计时：

```java
    Observable.interval(1, TimeUnit.SECONDS)
        .take(60)
        .map(aLong -> 59 - aLong)
        .subscribe(aLong -> Log.i(ARIRUS, "还剩：" + aLong));
log:
16:38:30.906  I/ARIURS: 还剩：59
16:38:31.907  I/ARIURS: 还剩：58
16:38:32.908  I/ARIURS: 还剩：57
...
16:39:27.912  I/ARIURS: 还剩：2
16:39:28.902  I/ARIURS: 还剩：1
16:39:29.903  I/ARIURS: 还剩：0
```

# 变换操作符

## Buffer GroupBy Window

- Buffer 定期收集Observable的数据放进一个数据包裹，然后发射这些数据包裹，而不是一次发射一个值。
- GroupBy 将一个Observable分拆为一些Observables集合，它们中的每一个发射原始Observable的一个子序列。
- Window 定期将来自原始Observable的数据分解为一个Observable窗口，发射这些窗口，而不是每次发射一项数据。

这三个操作符都是将原始数据分类处理，不过侧重点不一样：Buffer 按照原始数据顺序，将其打包成 List 发送到下游。GroupBy 则是根据发送数据进行分类，同一类的生成新的 GroupedObservable（也是一种 Observable），并将其发送到下游，这时数据流中数据不是 item 也不是 List<item> ，而是GroupedObservable<分类依据, item>。Window 则是位于二者之间，也是每 n 个 item 一组发送出去，发送的不是 List<item> ,而是 Observable<item>。

```java
    Observable<Integer> observable = Observable.range(1,9);

    observable.buffer(3).subscribe(new Action1<List<Integer>>() {
      @Override
      public void call(List<Integer> list) {
        Log.i(ARIRUS, "buffer call: "+list);
      }
    });

    observable.groupBy(new Func1<Integer, Boolean>() {
      @Override
      public Boolean call(Integer integer) {
        return integer%2==0;
      }
    }).subscribe(booleanIntegerGroupedObservable -> {
      if (booleanIntegerGroupedObservable.getKey())
        booleanIntegerGroupedObservable.subscribe(integer -> Log.i(ARIRUS, "groupBy 偶数 call: "+integer));
      else
        booleanIntegerGroupedObservable.subscribe(integer -> Log.i(ARIRUS, "groupBy 奇数 call: "+integer));
    });

    observable.window(3).subscribe(new Action1<Observable<Integer>>() {
      @Override
      public void call(Observable<Integer> integerObservable) {
        Log.i(ARIRUS, "window call: "+ integerObservable);
        integerObservable.subscribe(new Action1<Integer>() {
          @Override
          public void call(Integer integer) {
            Log.i(ARIRUS, "integerObservable call: "+integer);
          }
        });
      }
    });

log:
I/ARIURS: groupBy 奇数 call: 1
I/ARIURS: groupBy 偶数 call: 2
I/ARIURS: groupBy 奇数 call: 3
I/ARIURS: groupBy 偶数 call: 4
I/ARIURS: groupBy 奇数 call: 5
I/ARIURS: groupBy 偶数 call: 6
...
I/ARIURS: buffer call: [1, 2, 3]
I/ARIURS: buffer call: [4, 5, 6]
I/ARIURS: buffer call: [7, 8, 9]

I/ARIURS: window call: rx.subjects.UnicastSubject@3c6ccfb8
I/ARIURS: integerObservable call: 1
I/ARIURS: integerObservable call: 2
I/ARIURS: integerObservable call: 3
I/ARIURS: window call: rx.subjects.UnicastSubject@1eeb191
I/ARIURS: integerObservable call: 4
I/ARIURS: integerObservable call: 5
I/ARIURS: integerObservable call: 6
...
```

## Map FlatMap

- Map 原始Observable发射的每一项数据应用一个你选择的函数，然后返回一个发射
这些结果的Observable。
- FlatMap 使用一个指定的函数对原始Observable发射的每一项数据执行变换操作，这
个函数返回一个本身也发射数据的Observable，然后 FlatMap 合并这些Observables发射的数据，最后将合并后的结果当做它自己的数据序列发射。

这两个操作符，应该是大家用的最早的两个操作符，区别在于返回值不同，map 返回的是转变后的 item 数据，而 flatmap 则是返回转变后 item 的 Observable。例子很多就不写了。

# 过滤操作符

## ThrottleWithTimeout(Debounce) ThrottleFirst ThrottleLast(Sample)

- ThrottleFirst 发射采样期间的第一项数据。
- ThrottleLast 发射采样期间的最后一项数据。
- ThrottleWithTimeout 采样期间如果当前数据之后时间段内没有另外的数据，则发射当前数据。

三者的区别不是很大，差不多就是上述所描述的那样。
Debounce 在自定义搜索框时十分有用。

```java
    PublishSubject<CharSequence> publishSubject = PublishSubject.create();

    publishSubject.debounce(1,TimeUnit.SECONDS).subscribe(charSequence -> Log.i(ARIRUS, "EditText: "+charSequence));

    mEditText.addTextChangedListener(new TextWatcher() {
      @Override
      public void onTextChanged(CharSequence charSequence, int i, int i1, int i2) {
        publishSubject.onNext(charSequence);
      }

      ....
    });
```

这样的话，每次 edittext 的内容变化后，并且在1s内没有继续变化，则把其变化后的内容传输到下游。
ThrottleFirst 则可以用到按钮的消抖上面。

```java
    publishSubject.throttleFirst(1,TimeUnit.SECONDS).subscribe(integer -> Log.i(ARIRUS,"Button: "));
    mButton.setOnClickListener(view -> publishSubject.onNext(1));
```

## First Take(1)

- First 只发射第一项（或者满足某个条件的第一项）数据。
- Take 只发射前面的N项数据。

当 Take(1) 实质也是只发射第一项，但是二者最大的区别就是如果没有所谓的第一项，First 则会报错，原因是，First操作符是由 take(1).single() 实现的，single 操作符如果没有唯一的一个元素返回，就会抛出 NoSuchElementException , 而take本身不会抛出异常。

```java
    publishSubject.first().subscribe(Actions.empty(),Throwable::printStackTrace,()-> Log.i(ARIRUS,
        "finish"));
    mButton.setOnClickListener(view -> {
        publishSubject.onCompleted();
    });

log:
    java.util.NoSuchElementException: Sequence contains no elements
***********************************************************************
    publishSubject.take(1).subscribe(Actions.empty(),Throwable::printStackTrace,()-> Log.i(ARIRUS,
        "finish"));
    mButton.setOnClickListener(view -> {
        publishSubject.onCompleted();
    });
log:
    finish
```

Subject 没有向下游发射任何数据，只是发射了完成的通知，因此对于 first 直接报错，而 take 获取不到需要的数据，收到了完成的通知。同理 last() 和 takeLast() 操作符。

# 结合操作符

## CombineLatest Zip

- CombineLatest 当两个Observables中的任何一个发射了数据时，使用一个函数结合每个Observable发射的最近数据项，并且基于这个函数的结果发射数据。
- Zip 通过一个函数将多个Observables的发射物结合到一起，基于这个函数的结果为每个结合体发射单个数据项。

CombineLatest 操作符行为类似于 zip ，但是只有当原始的Observable中的每一个都发射了一条数据时 zip 才发射数据。 CombineLatest 则在原始的Observable中任意一个发射了数据时发射一条数据。当原始Observables的任何一个发射了一条数据时， CombineLatest 使用一个函数结合它们最近发射的数据，然后发射这个函数的返回值。

```java
    Observable<Long> longObservable1 = Observable.interval(1,TimeUnit.SECONDS).take(10);
    Observable<Long> longObservable2 = Observable.interval(3,TimeUnit.SECONDS).take(3);

    Observable.combineLatest(longObservable1,longObservable2,(aLong, aLong2) -> aLong+" "+aLong2)
        .subscribe(s -> Log.i(ARIRUS, "combineLatest: "+s));

    Observable.zip(longObservable1,longObservable2,(aLong, aLong2) -> aLong+" "+aLong2)
        .subscribe(s -> Log.i(ARIRUS, "zip: "+s));

log:
 I/ARIURS: combineLatest: 2 0
 I/ARIURS: combineLatest: 3 0
 I/ARIURS: combineLatest: 4 0
 I/ARIURS: combineLatest: 5 0
 I/ARIURS: combineLatest: 5 1
 I/ARIURS: combineLatest: 6 1
 I/ARIURS: combineLatest: 7 1
 I/ARIURS: combineLatest: 8 1
 I/ARIURS: combineLatest: 8 2
 I/ARIURS: combineLatest: 9 2

 I/ARIURS: zip: 0 0
 I/ARIURS: zip: 1 1
 I/ARIURS: zip: 2 2
```

这里有必要说明一下 combineLatest 没有 `0 0` `1 0` 这是由于此时 longObservable2 尚没有发射数据，因此只有一个 Observable 发射数据依然是没有可能向下游发射数据的。同时 zip 遵循两个 Observable 同步的数据的原则，因此不会出现 `X 0` 这种。

## Merge Concat

- Merge 合并多个Observables的发射物,可能会让合并的Observables发射的数据交错
- Concat 合并多个Observables的发射物,会让合并的Observables发射的数据保持原来的顺序

```java

    Observable<String> longObservable1 = Observable.interval(1,TimeUnit.SECONDS).take(5).map(aLong -> "ob1 "+aLong);

    Observable<String> longObservable2 = Observable.interval(3,TimeUnit.SECONDS).take(3).map(aLong -> "ob2 "+aLong);

    Observable.concat(longObservable1,longObservable2)
    .subscribe(string -> Log.i(ARIRUS, "concat: "+string));
log:
14:28:14.149 I/ARIURS: concat: ob1 0
14:28:15.100 I/ARIURS: concat: ob1 1
14:28:16.101 I/ARIURS: concat: ob1 2
14:28:17.092 I/ARIURS: concat: ob1 3
14:28:18.093 I/ARIURS: concat: ob1 4
14:28:21.106 I/ARIURS: concat: ob2 0
14:28:24.109 I/ARIURS: concat: ob2 1
14:28:27.112 I/ARIURS: concat: ob2 2
*******************************************************************
    Observable.merge(longObservable1,longObservable2)
    .subscribe(string -> Log.i(ARIRUS,"merge: "+string));

log:
14:30:21.293 I/ARIURS: merge: ob1 0
14:30:22.294 I/ARIURS: merge: ob1 1
14:30:23.295 I/ARIURS: merge: ob1 2
14:30:23.295 I/ARIURS: merge: ob2 0
14:30:24.286 I/ARIURS: merge: ob1 3
14:30:25.287 I/ARIURS: merge: ob1 4
14:30:26.288 I/ARIURS: merge: ob2 1
14:30:29.311 I/ARIURS: merge: ob2 2
```

# 其他操作符

## ToBlocking

- ToBlocking 要将一个 Observable 转换为一个 BlockingObservable。

BlockingObservable 会阻塞等待直到Observable发射了想要的数据，然后返回
这个数据（而不是一个Observable）。
由于在 Android 上不支持，链式操作，因此 List.forEach() 操作符不支持，想要遍历一个 Array 还是需要 for 语句，而我们可以使用 ToBlocking 来解决这个问题：

```java
void handleString(String string){//实现}
...
List<String> list = Arrays.asList("1", "s", "d", "43", "fg", "re");
Observable.from(list).toBlocking().subscribe(this::handleString);
```

PS:forEach 操作符在 BlockingObservable 和 Observable 都有，这里可以坐下对比：

```java
    Observable<Long> timeObservable = Observable.interval(1, TimeUnit.SECONDS, Schedulers.io()).take(5);

    Log.i(ARIRUS, "onViewCreated: start blocking forEach");

    timeObservable.toBlocking()
        .forEach(s -> Log.i(ARIRUS,
            "toBlocking forEach: " + s + " " + Thread.currentThread().getName()));
    Log.i(ARIRUS, "onViewCreated: start blocking subscribe");

    timeObservable.toBlocking()
        .subscribe(s -> Log.i(ARIRUS,
            "toBlocking subscribe: " + s + " " + Thread.currentThread().getName()));
    Log.i(ARIRUS, "onViewCreated: start foreach");

    timeObservable.take(5)
        .forEach(s -> Log.i(ARIRUS, "forEach: " + s + " " + Thread.currentThread().getName()));
    Log.i(ARIRUS, "onViewCreated: end");

log:
16:10:40.796 I/ARIURS: start blocking forEach
16:10:41.797 I/ARIURS: toBlocking forEach: 0 RxIoScheduler-2
16:10:42.798 I/ARIURS: toBlocking forEach: 1 RxIoScheduler-2
...
16:10:45.800 I/ARIURS: onViewCreated: start blocking subscribe
16:10:46.811 I/ARIURS: toBlocking subscribe: 0 main
16:10:47.802 I/ARIURS: toBlocking subscribe: 1 main
...
16:10:50.855 I/ARIURS: onViewCreated: start foreach
16:10:50.855 I/ARIURS: onViewCreated: end
16:10:51.856 I/ARIURS: forEach: 0 RxIoScheduler-2
16:10:52.857 I/ARIURS: forEach: 1 RxIoScheduler-2
...
```

这里可以看出 Observable.forEach() 是非阻塞的，BlockingObservable.forEach() 则是阻塞的。但是有一点我不是很明白，为什么 toBlocking.subscribe() 使得线程切换到了主线程。。

## RepeatWhen RetryWhen

RepeatWhen 和 RetryWhen 分别是 Repeat 与 Retry 的条件形式，就是说在条件满足的情况下才会进行重复/重试。二者主要的区别在于触发条件：

    repeat() resubscribes when it receives onCompleted().

    retry() resubscribes when it receives onError().

因此对于 RepeatWhen RetryWhen 两个操作符的参数分别是

    Func1<? super Observable<? extends Void>, ? extends Observable<?>>
    Func1<? super Observable<? extends Throwable>, ? extends Observable<?>> 
因为 onCompleted 没有参数，因此 Func1 的第一个形参是 Observable<? extends Void>，但是为什么是 Observable ？因为，其实是将每次 complete 通知，组成一个流，流上的数据就是一个个 complete 通知，并将每一个通知经过 handler 观察，决定是否继续重复，因此参数的形参名叫做 notificationHandler ,很直观了。同理 RetryWhen。还有一点需要注意：返回的 Observable 与 传输来的 Observable 必须是同一个，即链式规则不能打断。不然是无效的。

```java
    Observable.interval(1, TimeUnit.SECONDS)
        .take(5)
        .repeatWhen(new Func1<Observable<? extends Void>, Observable<?>>() {
          @Override
          public Observable<?> call(Observable<? extends Void> observable) {
            return observable.take(3);
          }
        })
        .subscribe(aLong -> Log.i(ARIRUS, "onViewCreated: " + aLong));
```

这里就是重复执行2次，带上本身总会执行一次，总共执行3次。

```java
    static int i = 1;
    Observable.interval(1, TimeUnit.SECONDS)
        .take(5)
        .repeatWhen(new Func1<Observable<? extends Void>, Observable<?>>() {
          @Override
          public Observable<?> call(Observable<? extends Void> observable) {
            return observable.flatMap(new Func1<Void, Observable<?>>() {
              @Override
              public Observable<?> call(Void aVoid) {
                if (i<3){
                  i++;
                  return Observable.just(1);
                }
                else
                  return Observable.error(new IOException());
              }
            });
          }
        })
        .subscribe(aLong -> Log.i(ARIRUS, "onViewCreated: " + aLong));
```

这样我们就可以根据当前情况来进行判断，是否需要重复执行。当参数 Observable 调用了 onComplete 或者 onError ，便会结束重复。还有一个小坑需要注意一下，这里调用的是 observable.flatMap() ,而非 observable.map(), 因为如果调用 map 返回 new IOException() ，相当于调用了 Observable.onNext() 方法，只有 flatMap 才是正确的写法。
使用 retryWhen 可以在失败时进行重新请求：

```java
    Observable.timer(3, TimeUnit.SECONDS)
        .repeatWhen(attempts -> attempts.zipWith(Observable.range(1, 3), (n, i) -> i)
            .flatMap(i -> Observable.timer(i, TimeUnit.SECONDS)))
        .subscribe(aLong -> Log.i(ARIRUS, "timer: "));
```

## ObserveOn SubscribeOn

- ObserveOn 指定一个观察者在哪个调度器上观察这个Observable，事件消费的线程
- SubscribeOn 指定Observable自身在哪个调度器上执行，事件产生的线程

简单理解的区别就是，SubscribeOn 是全局切换线程， ObserveOn 是局部切换线程。

```java
    Observable.just(1)
        .subscribe(integer -> Log.i(ARIRUS, "normal: " + Thread.currentThread().getName()));

    Observable.just(1)
        .map(integer -> {
          Log.i(ARIRUS, "subscribeOn map: "+ Thread.currentThread().getName());
          return  ""+i;
        })
        .doOnNext(integer -> Log.i(ARIRUS, "doOnNext subscribeOn: "+ Thread.currentThread().getName()))
        .subscribeOn(Schedulers.io())
        .subscribe(integer -> Log.i(ARIRUS, "subscribeOn: " + Thread.currentThread().getName()));

    Observable.just(1)
        .map(integer -> {
          Log.i(ARIRUS, "subscribeOn map: "+ Thread.currentThread().getName());
          return  ""+i;
        })
        .doOnNext(integer -> Log.i(ARIRUS, "doOnNext observeOn: "+ Thread.currentThread().getName()))
        .observeOn(Schedulers.io())
        .subscribe(integer -> Log.i(ARIRUS, "observeOn: " + Thread.currentThread().getName()));

log：
I/ARIURS: normal: main

I/ARIURS: subscribeOn map: RxIoScheduler-2
I/ARIURS: doOnNext subscribeOn: RxIoScheduler-2
I/ARIURS: subscribeOn: RxIoScheduler-2

I/ARIURS: subscribeOn map: main
I/ARIURS: doOnNext observeOn: main
I/ARIURS: observeOn: RxIoScheduler-3
```

我们使用第一条作为参照，开始线程为 main 第二条在 IO 线程上进行订阅，就是说从 item 发射，到最后的观察，全部是在 IO 线程上进行的。第三条，仅在最后观察的时候调用了 observeOn 方法，因此只是最后的观察是在 IO 线程上进行的，之前的操作 map doOnNext 依然是在 main 线程上进行的。

```java
    Observable.just(1)
        .subscribeOn(Schedulers.computation())
        .map(integer -> {
          Log.i(ARIRUS, "subscribeOn map: "+ Thread.currentThread().getName());
          return  ""+i;
        })
        .subscribeOn(Schedulers.newThread())
        .doOnNext(integer -> Log.i(ARIRUS, "doOnNext subscribeOn: "+ Thread.currentThread().getName()))
        .subscribeOn(Schedulers.io())
        .subscribe(integer -> Log.i(ARIRUS, "subscribeOn: " + Thread.currentThread().getName()));

    Observable.just(1)
        .observeOn(Schedulers.computation())
        .map(integer -> {
          Log.i(ARIRUS, "observeOn map: "+ Thread.currentThread().getName());
          return  ""+i;
        })
        .observeOn(Schedulers.newThread())
        .doOnNext(integer -> Log.i(ARIRUS, "doOnNext observeOn: "+ Thread.currentThread().getName()))
        .observeOn(Schedulers.io())
        .subscribe(integer -> Log.i(ARIRUS, "observeOn: " + Thread.currentThread().getName()));

log:
I/ARIURS: subscribeOn map: RxComputationScheduler-1
I/ARIURS: doOnNext subscribeOn: RxComputationScheduler-1
I/ARIURS: subscribeOn: RxComputationScheduler-1

I/ARIURS: observeOn map: RxComputationScheduler-2
I/ARIURS: doOnNext observeOn: RxNewThreadScheduler-2
I/ARIURS: observeOn: RxIoScheduler-3
```

这两条分别是在每个操作符之前分别多次调用了 subscribeOn 与 observeOn ，可以看到 subscribeOn 只有在第一次调用时其起作用，之后调用是不会改变其订阅线程的，而 observeOn 则是每次调用时，都会起作用。

多次调用 subscribeOn 仅在调用 doOnSubscribe 情况下才有用：

```java
    Observable.just(1)
        .subscribeOn(Schedulers.computation())
        .map(integer -> {
          Log.i(ARIRUS, "subscribeOn map: "+ Thread.currentThread().getName());
          return  ""+i;
        })
        .doOnSubscribe(()-> Log.i(ARIRUS, "doOnSubscribe 1: " + Thread.currentThread().getName()))
        .subscribeOn(Schedulers.newThread())
        .doOnNext(integer -> Log.i(ARIRUS, "doOnNext subscribeOn: "+ Thread.currentThread().getName()))
        .doOnSubscribe(()-> Log.i(ARIRUS, "doOnSubscribe 2: " + Thread.currentThread().getName()))
        .subscribeOn(Schedulers.io())
        .subscribe(integer -> Log.i(ARIRUS, "subscribeOn: " + Thread.currentThread().getName()));

log:
I/ARIURS: doOnSubscribe 2: RxIoScheduler-2
I/ARIURS: doOnSubscribe 1: RxNewThreadScheduler-1
I/ARIURS: subscribeOn map: RxComputationScheduler-1
I/ARIURS: doOnNext subscribeOn: RxComputationScheduler-1
I/ARIURS: subscribeOn: RxComputationScheduler-1
```

可以看到调用 doOnSubscribe ，如果后面还有调用 subscribeOn 则 doOnSubscribe 工作在此线程。因此多次调用 subscribeOn 可以改变开始订阅的线程，不过整体的事件的生产线程不变依然还是只有第一个 subscribeOn 线程有效。

# 结尾

本篇中，我们对比并列举了一些常用的操作符，并且针对某些操作符给出了一些 demo。下一篇，我们将会继续进行操作符的进阶分析，并从源码的角度分析操作符的原理。