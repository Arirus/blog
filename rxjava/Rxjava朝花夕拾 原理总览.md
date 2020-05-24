---
date : 2018-11-29T14:31:59+08:00
tags : ["Rxjava", "Android", "用法"]
title : "Rxjava朝花夕拾 原理总览"

---

本次源码分析是以 GitHub 上 RxJava 仓库 [2.1.1](https://github.com/ReactiveX/RxJava)版本来进行的分析，同时以[官方中文文档](https://legacy.gitbook.com/book/mcxiaoke/rxdocs/details)作为参考。

上一篇中，我们着重的对 Rxjava 的应用进行了分析，给出了一些笔者的能想到的常用的例子，相信针对日常的使用应该是足够的。那么本篇作为 Rxjava 系列的最后一篇，从整体流程上来分析下 Rxjava 的工作流程，理解下 Rxjava 设计思路。

<!--more-->

# Rxjava 流程
## 基本流程
首先我们来看看一行最基本的 Rxjava 代码：
```java
mDisposable = Single.just(1).subscribe(integer -> Log.i("RxjavaActivity", "Arirus onCreate: "+integer));
```
很简单，上游发送一个 1 ，下游接受时把其打印出来。上游发送端使用 `Single.just` 方法来生成被观察者，我们跟进去看看是怎样的一个逻辑：
```java
public static <T> Single<T> just(final T item) {
    ObjectHelper.requireNonNull(item, "value is null");
    return RxJavaPlugins.onAssembly(new SingleJust<T>(item));
}

public static <T> Single<T> onAssembly(@NonNull Single<T> source) {
    Function<? super Single, ? extends Single> f = onSingleAssembly;
    if (f != null) {
        return apply(f, source);
    }
    return source;
}
```
这里是直接调用了 RxJavaPlugins.onAssembly 方法，这个方法其实就是用了一个全局变量 onSingleAssembly 来对数据源进行转换，当我们没有设置的时候其实 just 方法是直接返回了一个 SingleJust 对象。

好了，我们再看看 SingleJust 是一个怎么实现：
```java
public final class SingleJust<T> extends Single<T> {

    final T value;

    public SingleJust(T value) {
        this.value = value;
    }

    @Override
    protected void subscribeActual(SingleObserver<? super T> observer) {
        observer.onSubscribe(Disposables.disposed());
        observer.onSuccess(value);
    }

}
```
实现也很简单，继承于 Single 对象，我们知道 Single 对象主要就是这么几类方法：一类是各种生成方法，create just contact 等，一类就是各种操作符 map flatmap bufer 等，还有一类就是 subscribe 方法。最后一类方法其实最后调用这么一句话：
```java
public final void subscribe(SingleObserver<? super T> observer) {
    ObjectHelper.requireNonNull(observer, "subscriber is null");
    ...
    try {
        subscribeActual(observer);
    } catch (NullPointerException ex) {
        throw ex;
    } catch (Throwable ex) {
        ...
    }
}
```
因此实际上在 subscribe 时，调用是每个 Single 对象的 subscribeActual 方法，subscribeActual 才是出来逻辑的核心。还回到 SingleJust ，其本身 subscribeActual 只有两句话:
```java
observer.onSubscribe(Disposables.disposed());
observer.onSuccess(value);
```
就是告诉下游：我开始发送数据了；我发送这个数据了；还有潜台词我这次不会发送失败。很有意思在告诉下游我开始发送数据时，返回了一个已经丢弃的可丢弃对象：
```java
/**
    * Returns a disposed Disposable instance.
    * @return a disposed Disposable instance
    */
@NonNull
public static Disposable disposed() {
    return EmptyDisposable.INSTANCE;
}
```
这是由于就一个数据需要发送给下游，发送完了就完事儿了，同时也是完全不会出错的，所以直接返回一个不可以再次丢弃的可丢弃对象。这就是最最简单一次 Rxjava 的发送的过程。下面我们看看如果和操作符连用会是怎样的结果。

## 调用 Map 操作符
代码很简单我们只需要在 Just 之后调用一个 map 操作符：
```java
mDisposable = Single.just(1).map(integer -> integer>0).subscribe(bool -> Log.i("RxjavaActivity", "Arirus onCreate: "+bool));
```
这里我们将 Integer 转换成了 Boolean 类型数据。来看看 map 操作符干了什么事儿：
```java
public final <R> Single<R> map(Function<? super T, ? extends R> mapper) {
    ObjectHelper.requireNonNull(mapper, "mapper is null");
    return RxJavaPlugins.onAssembly(new SingleMap<T, R>(this, mapper));
}
```
类似 Just 方法，我们直接看 SingleMap 就好了：
```java
public final class SingleMap<T, R> extends Single<R> {
    final SingleSource<? extends T> source;

    final Function<? super T, ? extends R> mapper;

    public SingleMap(SingleSource<? extends T> source, Function<? super T, ? extends R> mapper) {
        this.source = source;
        this.mapper = mapper;
    }

    @Override
    protected void subscribeActual(final SingleObserver<? super R> t) {
        source.subscribe(new MapSingleObserver<T, R>(t, mapper));
    }

    static final class MapSingleObserver<T, R> implements SingleObserver<T> {

        final SingleObserver<? super R> t;

        final Function<? super T, ? extends R> mapper;

        MapSingleObserver(SingleObserver<? super R> t, Function<? super T, ? extends R> mapper) {
            this.t = t;
            this.mapper = mapper;
        }

        @Override
        public void onSubscribe(Disposable d) {
            t.onSubscribe(d);
        }

        @Override
        public void onSuccess(T value) {
            R v;
            try {
                v = ObjectHelper.requireNonNull(mapper.apply(value), "The mapper function returned a null value.");
            } catch (Throwable e) {
                Exceptions.throwIfFatal(e);
                onError(e);
                return;
            }

            t.onSuccess(v);
        }

        @Override
        public void onError(Throwable e) {
            t.onError(e);
        }
    }
}
```
其实这里有一点十分值得我们说下：
**使用操作符链接被观察者，依然会生一个新的被观察者，并不是只有创建操作符会创建新的被观察者**。同时创建的时候还将自己也传入到了新的 Single 对象当中了。这其实就是说明，Rxjava 不是使用的 Builder 模式一直在修改上游的被观察者，而是类似于装饰者模式将一个个的被观察者相关联起来。明白这点对于理解 Rxjava 的工作流程很重要，我们继续向下看。SingleMap.subscribeActual 方法，时上游对象（SingleJust）订阅了一个 MapSingleObserver（继承于 SingleObserver 的观察者）。这样其实就是在 SingleMap 搞了一套订阅流程：
```java
source.subscribe(new MapSingleObserver<T, R>(t, mapper));
```
将下游的 SingleObserver 以参数的形式传入到了 MapSingleObserver 对象当中，最后上游的数据在订阅时都会通过 MapSingleObserver 调用 onSuccess 方法传入到下游的 onSuccess 方法中。

到这里我们其实差不多可以理清 Rxjava 的大体思路。之前我们说过这种`冷的`被观察者只有在被订阅的时候才会向下游发送数据，所以是订阅前的最后一个被观察者发起 subscribe 操作，调用了上游的被观察者的 subscribe 方法，这样依次向上传递直到最上游部分，在我们这里就是 SingleJust 对象，然后最上游的开始发射数据（onSuccess）到下游，一路上所有的观察者（SingleObserver）都会调用到 onSuccess 方法，正好到 MapSingleObserver.onSuccess 将数据进行了映射处理（mapper.apply）同时向更下游发出，直到最后的 SingleObserver.onSuccess 就是本例中的 Consumer 。因此 Rxjava 的整体流程其实是**最下游的被观察者依次将上游的被观察的开关打开，直到最上游被观察发送数据，依次发送给下游的观察者，最终传递给最后一个观察者**。整个过程类似了 OkHttp 在请求时使用的 Interceptor，而 subscribe 与 onSubscribe 正好是上行和下行两个对应的方法。

![Rxjava 原理](/img/Rxjava_Principle.png)

# 线程切换
线程切换绝对是 Rxjava 能广为流传的原因之一，我们知道 Rxjava 中，subscribeOn 用来设定订阅线程，observeOn 用来设定观察线程，这两个方法是切换线程的关键所在，那我们就分别来看看这两个方法中都是怎样的一个实现。首先看看 subscribeOn 实现，我们进行如下调用：
```java
Single.just(1)
        .subscribeOn(Schedulers.newThread())
        .subscribe(aBoolean -> Log.i("RxjavaActivity", "Arirus onCreate: " + aBoolean));
```

## subscribeOn
subscribeOn 用来设定订阅线程，所谓订阅线程就是事件源产生的线程。subscribeOn 其实质内容也是生成一个 Single 对象，就是上面说的 SingleJust 和 SingleMap 一样，这里我们都能直接脑补代码：肯定是在 subscribeActual 里面又自己实现了一个 SingleObserver 并且和上游的被观察者通过 subscribe 方法关联了起来。那么我们直接看 subscribeActual 方法和其自己定义的 SingleObserver 对象。
```java
protected void subscribeActual(final SingleObserver<? super T> observer) {
    final SubscribeOnObserver<T> parent = new SubscribeOnObserver<T>(observer, source);
    observer.onSubscribe(parent);
    Disposable f = scheduler.scheduleDirect(parent);
    parent.task.replace(f);
}

static final class SubscribeOnObserver<T>
extends AtomicReference<Disposable>
implements SingleObserver<T>, Disposable, Runnable {
    ...
    SubscribeOnObserver(SingleObserver<? super T> actual, SingleSource<? extends T> source) {
        this.downstream = actual;
        this.source = source;
        this.task = new SequentialDisposable();
    }

   ...
}
```
我猜到了开头，却没有猜到结尾。虽然 SubscribeOnObserver 作为了一个 SingleObserver 确实存在，但是并没有和上游数据直接通过 subscribe 方法直接联系起来。那我们来仔细看看 SubscribeOnObserver 是怎样的实现。

### SubscribeOnObserver
SubscribeOnObserver 继承了 Disposable 的原子引用并且实现了 SingleObserver，Disposable，Runnable 的接口。第一个奇怪的点：就是说它既是一个观察者（SingleObserver）也是一个可丢弃对象（Disposable）。另一个奇怪的点是：SubscribeOnObserver 继承了 AtomicReference<Disposable> 但是构造函数中并没有传入任何 Disposable 实例，因此 AtomicReference.value 是一个 null 值。

我们继续向下看，subscribeActual 方法中调用了 observer.onSubscribe(parent)，其中 parent 就是 SubscribeOnObserver 对象，此时充当了 Disposable 的角色：
```java
public void ConsumerSingleObserver.onSubscribe(Disposable d) {
    DisposableHelper.setOnce(this, d);
}
```
observer 就是一个 ConsumerSingleObserver 对象，其实现类似 SubscribeOnObserver，继承于 AtomicReference<Disposable>，但是构造函数中也没有设置 AtomicReference.value 值，直接调用了 DisposableHelper.setOnce 方法：
```java
public static boolean setOnce(AtomicReference<Disposable> field, Disposable d) {
    ObjectHelper.requireNonNull(d, "d is null");
    if (!field.compareAndSet(null, d)) {
        ...
    }
    return true;
}
```
直接将 d 替换了自己的 null value，因此就像是相当于赋值了。接着就是 scheduler.scheduleDirect 方法：
```java
public Disposable scheduleDirect(@NonNull Runnable run, long delay, @NonNull TimeUnit unit) {
    final Worker w = createWorker();

    final Runnable decoratedRun = RxJavaPlugins.onSchedule(run);

    DisposeTask task = new DisposeTask(decoratedRun, w);

    w.schedule(task, delay, unit);

    return task;
}
```
由于我们是在 Schedulers.newThread 进行订阅的，我们直接看 NewThreadScheduler.createWorker 方法。
```java
public Worker NewThreadScheduler.createWorker() {
    return new NewThreadWorker(threadFactory);
}

public NewThreadWorker(ThreadFactory threadFactory) {
    executor = SchedulerPoolFactory.create(threadFactory);
}
```
因此我们可以知道 **Worker 包含了一个线程池，用来执行后台方法**。RxJavaPlugins.onSchedule 照例跳过，decoratedRun 就是 run 对象。
DisposeTask 将 Worker 和 Runnable 封装了起来，我们大致可以猜测就是要在这个 Worker 提供的线程上执行 Runnable 了。最后调用了 NewThreadWorker.scheduleActual 方法：
```java
public ScheduledRunnable scheduleActual(final Runnable run, long delayTime, @NonNull TimeUnit unit, @Nullable DisposableContainer parent) {
    ...
    ScheduledRunnable sr = new ScheduledRunnable(decoratedRun, parent);
    ...
    Future<?> f;
    try {
        if (delayTime <= 0) {
            f = executor.submit((Callable<Object>)sr);
        } else {
            f = executor.schedule((Callable<Object>)sr, delayTime, unit);
        }
        sr.setFuture(f);
    } catch (RejectedExecutionException ex) {
        ...
    }

    return sr;
}
```
很清晰，就是将 Runnable 放到 executor 进行执行，最后返回 DisposeTask 对象（实现了 Disposable 接口）。简单的描述这里的步骤就是：

- 创建执行的线程池
- 创建 Runnable ，并将实际工作的 Runnable 当作参数传入（SubscribeOnObserver）
- 将 Runnable 放到执行的线程池中进行执行
- 返回一个可取消对象

最后，在 subscribeActual 中将返回的 Disposable 赋值给 SubscribeOnObserver 对象。

好的，那我们从整体上说一下 SingleSubscribeOn.subscribeActual 流程：

- SubscribeOnObserver 是一个代理 Disposable 和下游的 SingleObserver 联系起来，之后如果要关闭 Rxjava 流就是关闭这个代理  Disposable 对象；
- SubscribeOnObserver 同时又是一个 SingleObserver，用来向下游 SingleObserver 传递数据；
- SubscribeOnObserver 同时又是一个 Runnable，用来在线程池上执行操作；
- SubscribeOnObserver 内部又有一个 SequentialDisposable，用来控制上游的事件流；
- 上游的事件流是将 SubscribeOnObserver 运行在线程池中时进行请求产生的 ：

```java
@Override
public void SubscribeOnObserver.run() {
    source.subscribe(this);
}
```
之后的数据响应都是运行在线程池中的线程中的，一个简单流程图如下：

![subscirbeOn](/img/Rxjava_subscirbeOn.png)

这也是为什么，只有第一个 subscribeOn 才能使得订阅线程生效，因为他在逻辑订阅流程中的最后一环，之前的都不算数。如果两个 subscribeOn 之间有 doOnSubscribe 方法，总会是和后一个 subscribeOn 线程保持一致，因为后一个其实是属于逻辑订阅流程的前一个。所以 subscribeOn 大致就是这样一个流程：**自己做个代理 Observer 给下游发送数据，同时在另一个线程中向上游请求数据**，除此之外什么也不干。

## observeOn
observeOn 用来设置观察线程，可以理解成异步回调的响应线程。理解了 subscribeOn 之后，应该对于理解 observeOn 会好很多。直接看 SingleObserveOn 代码：
```java
protected void SingleObserveOn.subscribeActual(final SingleObserver<? super T> observer) {
    source.subscribe(new ObserveOnSingleObserver<T>(observer, scheduler));
}
```
熟悉的套路，上游和 ObserveOnSingleObserver 建立了联系，这次因为是观察，因此不需要代理，我们直接看 ObserveOnSingleObserver 是怎么实现的：
```java
static final class ObserveOnSingleObserver<T> extends AtomicReference<Disposable>
    implements SingleObserver<T>, Disposable, Runnable {
    ...

    ObserveOnSingleObserver(SingleObserver<? super T> actual, Scheduler scheduler) {
        this.downstream = actual;
        this.scheduler = scheduler;
    }

    ...
    @Override
    public void onSuccess(T value) {
        this.value = value;
        Disposable d = scheduler.scheduleDirect(this);
        DisposableHelper.replace(this, d);
    }

    @Override
    public void onError(Throwable e) {
        this.error = e;
        Disposable d = scheduler.scheduleDirect(this);
        DisposableHelper.replace(this, d);
    }

    @Override
    public void run() {
        Throwable ex = error;
        if (ex != null) {
            downstream.onError(ex);
        } else {
            downstream.onSuccess(value);
        }
    }

    ...
}
```
很清晰了：**上游发来数据，将线程切换为新的线程，在新线程中将数据发送给下游**

![observeOn](/img/Rxjava_observeOn.png)

# Disposable 

Disposable 是一个开关，通常管理两个部分：

- 上游事件发送是否结束
- 如果有线程切换，那么线程的关闭

这里我们通过 Observable 的例子来看看 Disposable 具体的作用：
```java
Observable.range(1,5).subscribeOn(Schedulers.newThread()).subscribe(new Observer<Integer>() {
    @Override public void onSubscribe(Disposable d) {
    mDisposable = d;
    }

    @Override public void onNext(Integer integer) {
    if (integer == 3) mDisposable.dispose();
    }

    @Override public void onError(Throwable e) {

    }

    @Override public void onComplete() {

    }
});
```
我们在 Rxjava2.X 那篇说过，Disposable 相当于1.0的 Subscription，在订阅的时候会传递给观察者。这里我们就通过 onSubscribe 方法来获取到 Disposable 对象。前面的流程都熟了，直接看看 ObservableSubscribeOn.subscribeActual 方法：
```java
public void ObservableSubscribeOn.subscribeActual(final Observer<? super T> observer) {
    final SubscribeOnObserver<T> parent = new SubscribeOnObserver<T>(observer);

    observer.onSubscribe(parent);

    parent.setDisposable(scheduler.scheduleDirect(new SubscribeTask(parent)));
}
```
极类似 SingleSubscribeOn.subscribeActual ，方法 scheduler.scheduleDirect 返回 Disposable 对象，直接赋值给了 SubscribeOnObserver 。同时我们知道 SubscribeOnObserver 就是下游收到的 Disposable 对象。当调用 Disposable.dispose 方法时：
```java
public void SubscribeOnObserver.dispose() {
    DisposableHelper.dispose(upstream);
    DisposableHelper.dispose(this);
}
```
首先调用了上游 Disposable.dispose 告诉他不要再发送数据了，本例中使用的是 range 方法，那么看下方法内部：
```java
static final class RangeDisposable extends BasicIntQueueDisposable<Integer>{
    ...
    void run() {
        ...
        for (long i = index; i != e && get() == 0; i++) {
            actual.onNext((int)i);
        }
        if (get() == 0) {
            lazySet(1);
            actual.onComplete();
        }
    }
    ...
    public void dispose() {
        set(1);
    }
}
```
RangeDisposable 也是原子操作的，收到 dispose 后，会直接修改 Integer 的值，从而结束发送数据。

其次调用了本身的 dispose ，上面我们说过了 scheduler.scheduleDirect 返回的是一个 DisposeTask 对象，因此会调用 DisposeTask.dispose 方法：
```java
public void DisposeTask.dispose() {
    if (runner == Thread.currentThread() && w instanceof NewThreadWorker) {
        ((NewThreadWorker)w).shutdown();
    } else {
        w.dispose();
    }
}
```
直接将线程池关闭。这样 Disposable 两个作用就都达到了。

# 小结
本篇我们通过几个简单的例子来说明了 Rxjava 的原理是怎样的，像链式调用一样一个个的被观察者和观察者依次链接起来，每一个操作符完成自己生成，转换或者切换工作。分析了 subscribeOn 与 observeOn 的工作原理，最后分析了 Disposable 是怎么既可以关闭上游数据，也可以关闭线程池的，本系列的内容就先告一段落了，想到什么再补充。