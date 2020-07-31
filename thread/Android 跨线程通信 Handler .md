---
date : 2018-12-14T17:18:12+08:00
tags : ["底层", "Android", "跨进程通信"]
title : "Android 跨线程通信 Handler"

---
上一篇，我们了解了 Java 中，关于多线程的一些基本概念，和常规用法。本篇来看看 Android 的进程间通信，虽然 Android 也是由 Java 来开发的，但是我们很少使用 wait/notify 机制在进程通信当中，原因很好理解：你不可能让主线程阻塞等待，当进行网络请求的时候。同时，大多数情况下，在工作线程完成工作后会需要给主线程进行通知，因此简单的使用 Java 的线程间同步的机制肯定是不能满足 Android 的需求，因此 Handler 就是为了解决 Android 的线程间通信而设计的。

<!--more-->

# Handler 模型概览
我们知道 Android 既不允许在主线程中访问网络，也不允许在非子线程中对 UI 进行修改，其实这一切都是为了最大限度保证 UI 的流畅，Android 大约每 16ms 会刷新一次，因此不允许在主线程进行网络访问，这可能会耗时很长事件。同样 Android 的绘制都是在主线程中进行的，这是由于 UI 的绘制是线程不安全的，因此其底层其实是没有加锁的，这样做也是处于性能的考虑，上一篇中，我们对比了使用 synchronized 关键字，时间消耗要比没有使用时提升了60倍，可见 synchronized 对于效率有着极高的影响。所以 Handler 最最常见的应用就是在子线程通知 UI 线程进行更新，我们先看一个最简单的实现例子：
```java
public void onClick(View view) {
    new Thread(() -> {
        Log.i("HandlerActivity", "onClick");
        SystemClock.sleep(5000);
        Message message = Message.obtain();
        message.obj = "Download Finish！";
        mHandler.sendMessage(message);
    }).start();
}

Handler mHandler = new Handler() {
    @Override
    public void handleMessage(Message msg) {
      super.handleMessage(msg);
      Log.i("HandlerActivity",
          "handleMessage: " + msg.obj.toString() + " " + Thread.currentThread().getName());
    }
};


log:
25016-25100 I/HandlerActivity: onClick
25016-25016 I/HandlerActivity: handleMessage: Download Finish！ main
```
在执行 onClick 函数时，会启动一个线程来模拟下载任务，在下载任务完成之后，会通知主线程。这就是一个最简单的进程间通信。其实，Handler 的机制也很好理解。我们根据下图进行简要的说明：

![Handler](/img/Handler.png)

- Handler Looper MessageQueue 是实现 Handler 机制的基石
- Handler 用于发送一个 Message 到 MessageQuene 中或者接收一个 Message 
- Looper 处于无限循环状态，从 MessageQueue 中取出 Message
- MessageQueue 负责自己内部的 Message 的有序排列

这个模型很好理解，就像日本的旋转寿司，这里我们就不多做说明了。

# Hanlder
上面说到 Handler 负责 Message 的发送与接受，这里我们我就从 Handler 入手来看看其实现：
```java
public Handler(Callback callback, boolean async) {
    ...
    mLooper = Looper.myLooper();
    if (mLooper == null) {
        throw new RuntimeException(
            "Can't create handler inside thread that has not called Looper.prepare()");
    }
    mQueue = mLooper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
```
其构造函数中，总是会先尝试获取一个 Looper，当 Looper 为null时，就会抛出异常，但是我们上面的代码并没有抛出异常，原因是我们的 Handler 是主线程中的 Handler，而主线程的 Looper 是早早的就已经初始化好了，因此不会抛出异常。

而这一切都是由 ThreadLocal 里面获取的。主线程的Looper是在 ActivityThread 的 main 函数中创建的，同时也是这时候把 Looper 存储到 ThreadLocal 里面的。 

同时消息队列 MessageQueue 也是与 Looper 相绑定的，此时也与 Handler 相绑定，因此可以这么说，**一个线程中，有且只有 Looper，每个 Looper 上只有一个 MessageQueue 与其绑定**。

## 发送消息
Handler 发送消息也很简单，Handler 支持发送 Message，也支持发送一个 Runnable，其实后者最后也是被封装成一个 Message 来进行发送的：
```java
//Handler

public final boolean sendMessage(Message msg)
{
    return sendMessageDelayed(msg, 0);
}
...
public final boolean post(Runnable r)
{
    return  sendMessageDelayed(getPostMessage(r), 0);
}
...
private static Message getPostMessage(Runnable r) {
    Message m = Message.obtain();
    m.callback = r;
    return m;
}
...
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    msg.target = this;
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```
所以最终，就是 Handler 将 Message 发送到 MessageQueue 上，完成了消息的发送。

### enqueueMessage
好的 我们来看看，在消息队列上，message 是如何入队的：
```java
boolean enqueueMessage(Message msg, long when) {
    // 一个 msg 的targt是必须要有的
    // 通过 Handler.enqueueMessage 方法我们可以知道
    // 每个 msg 在发射的时候都会 设置一个 Handler 作为其 target
    if (msg.target == null) {
        throw new IllegalArgumentException("Message must have a target.");
    }
    if (msg.isInUse()) {
        throw new IllegalStateException(msg + " This message is already in use.");
    }

    synchronized (this) {
        if (mQuitting) {
            IllegalStateException e = new IllegalStateException(
                    msg.target + " sending message to a Handler on a dead thread");
            Log.w(TAG, e.getMessage(), e);
            msg.recycle();
            return false;
        }

        msg.markInUse();
        msg.when = when;
        Message p = mMessages;
        boolean needWake;
        if (p == null || when == 0 || when < p.when) {
            // New head, wake up the event queue if blocked.
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked;
        } else {
            // Inserted within the middle of the queue.  Usually we don't have to wake
            // up the event queue unless there is a barrier at the head of the queue
            // and the message is the earliest asynchronous message in the queue.
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;
            for (;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    break;
                }
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
            msg.next = p; // invariant: p == prev.next
            prev.next = msg;
        }

        // We can assume mPtr != 0 because mQuitting is false.
        if (needWake) {
            nativeWake(mPtr);
        }
    }
    return true;
}
```

## 接受消息
除了发送 Message，Handler 还要处理接收的消息：
```java
public void dispatchMessage(Message msg) {
    if (msg.callback != null) {
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        handleMessage(msg);
    }
}
```
在 Looper 中，每次接受到消息，都会将 Message 分发下来。Handler 接收 Message 时会判断其内部是否封装了 Runnable，如果封装了则会直接调用其 run 方法。否则会调用 handleMessage 方法，而 handleMessage 方法就是我们每次 Handler 时会重写的方法。这样一个 Message 的接受便结束了。

## 内存泄漏
细心的同学在使用 Handler 时，一定会发现会 IDE 会发出，**可能会出现内存泄漏的警告**，那我们就简单的分析下，为什么会出现内存泄漏。
我们在实现 Handler 时，是以 Activity 的匿名内部类的形式实现的，其持有一个外部类的引用（Activity 的引用），如果我们以延时的形式来传送一个 Message 时，Message 会进入到 MessageQueue 中，延时结束时才会分发到 Handler 中，但如果在这个过程中，Activity Destroy 是无法被释放内存的，这样其实会造成内存泄漏。
```java
mHandler.postDelayed(new Runnable() {
    @Override
    public void run() {
        Log.i("HandlerActivity", "Arirus run: RUN！！！！！！！");
    }
},1000*30);

log:
HandlerActivity: Arirus run: RUN！！！！！！！
```
这里我们发送 Message 时，进行了一个 30 秒的延时。之后，我们退出 Activity 并手动 GC 一下，我们来看看内存分析：

![handler_leak.png](/img/handler_leak.png)

确实是由于 Handler 的存在，而造成了 HandlerActivity 无法被回收，造成了内存泄漏。30s 之后，所有的 Message 都已经被消化完毕，这时 HandlerActivity 才会被回收。

![handler_no_leak.png](/img/handler_no_leak.png)

所以，对于在 Activity 中直接使用了匿名内部类，一定要小心内存泄漏的风险，不过万幸，这种泄漏类似于 Disposable 没有 dispose 一样，当完成后会继续释放，但是对于这种不可控的操作尽量还是避免。同样这也是为什么 Handler 的构造函数提供了一个 CallBack 参数的构造函数，用来避免直接实现一个内部子类。在 dispatchMessage 方法中，如果 mCallback 存在，会直接调用，就无需实现 handleMessage 方法。不过这样还是会有一个 Handler.CallBack 的匿名内部类，所以依然可能会出现内存泄漏。所以最正确的做法应该是**定义一个静态内部类或者外部类实现 Handler ，同时将 Activity 作为弱引用和 Handler 绑定在一起，当 Handler 收到 Message 时判断 Activity 的状态，再进行相应的操作**

![handler_custom.png](/img/handler_custom.png)

以上就是 Handler 的一些基本操作，这个也是 Android 开发组直接暴漏给开发者的类，那么下面我们就看看 Handler 模型的核心 Looper。

# Looper
Looper 扮演者消息循环的角色，负责从 MessageQueue 中不断的获取新的 Message。我们在 [Android 组件 IntentService 与 JobIntentService](https://arirus.cn/post/android-%E7%BB%84%E4%BB%B6-intentservice-%E4%B8%8E-jobintentservice-/) 曾对 Looper 的 loop 方法进行了分析，这里就不再分析了。
但是我们要看看 Looper 另一些关键方法：
```java
public static void prepare() {
    prepare(true);
}
private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
}
public static void prepareMainLooper() {
    prepare(false);
    synchronized (Looper.class) {
        if (sMainLooper != null) {
            throw new IllegalStateException("The main Looper has already been prepared.");
        }
        sMainLooper = myLooper();
    }
}
```
首先是 prepare 三连，我们知道在 Handler 创建时会判断当前线程的 Looper 是否已经准备完毕。因此调用的就是 prepare 方法，不过对于主线程比较特别，Looper 是不能退出的，其次 MainLooper 需要在任何线程中都能获得，因此使用了一个静态变量来将 Looper 保存了起来。这里需要注意的一点是使用了 ThreadLocal 静态实例来对 Looper 进行保存。我们知道，线程不同于进程，所有线程是共享同一个进程中的内存数据，ThreadLocal 顾名思义就是保存线程的本地局部变量，就是说每个线程中的保存的变量都是相互独立的，因此 ThreadLoacl 中的数据是不会进行线程间共享的，每个线程对于一个唯一的 ThreadLocal 内部数据。其内部实现也说明了这一点：
```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}

public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
```
ThreadLocalMap 是和当前所在的线程相关联的，所以不同线程获取的 ThreadLocalMap 不是同一个，因此可以把每个线程中的局部变量保存了起来而不冲突。因此这里使用 ThreadLocal 来保存每个线程中的 Looper 是再合适不过了。当初始化结束之后，就可以获取 Looper 开启循环了，分析可以看上面提到的[Android 组件 IntentService 与 JobIntentService](https://arirus.cn/post/android-%E7%BB%84%E4%BB%B6-intentservice-%E4%B8%8E-jobintentservice-/)。

# MessageQueue
MessageQueue 是插入和读取 Message 的单链表，插入我们就不看了，我们重点关注下读取：
```java
Message next() {
    // Return here if the message loop has already quit and been disposed.
    // This can happen if the application tries to restart a looper after quit
    // which is not supported.
    for (;;) {
        if (nextPollTimeoutMillis != 0) {
            Binder.flushPendingCommands();
        }

        nativePollOnce(ptr, nextPollTimeoutMillis);

        synchronized (this) {
            // Try to retrieve the next message.  Return if found.
            final long now = SystemClock.uptimeMillis();
            Message msg = mMessages;
            ...
            if (msg != null) {
                ...
                return msg;
            } 
            ...
            // Process the quit message now that all pending messages have been handled.
            if (mQuitting) {
                dispose();
                return null;
            }
        }
    }
}
```
之前在 Looper.loop 方法中，我们看到了 MessageQueue.next 会阻塞住线程，其实阻塞就是由于 nativePollOnce 方法造成的线程挂起，当有新的 Message 发送过来，便会继续执行，当需要退出时，会直接返回一个 null 给 Looper，Looper 便也会退出。

# HandlerThread
HandlerThread 就是拥有 Handler 的线程，我们可以通过其 Looper 来对线程进行操作。
[Android 组件 IntentService 与 JobIntentService](https://arirus.cn/post/android-%E7%BB%84%E4%BB%B6-intentservice-%E4%B8%8E-jobintentservice-/)中已经分析了 HandlerThread 的源码，大部分东西都已经说明，这里补充一点：
```java
public void run() {
    Looper.prepare();
    synchronized (this) {
        mLooper = Looper.myLooper();
        notifyAll();
    }
    Process.setThreadPriority(mPriority);
    Looper.loop();
}
public Looper getLooper() {    
    // If the thread has been started, wait until the looper has been created.
    synchronized (this) {
        while (isAlive() && mLooper == null) {
            try {
                wait();
            } catch (InterruptedException e) {
            }
        }
    }
    return mLooper;
}
```
这里的 notifyAll/wait 方法，就是为了**保证先生成一个 Looper 实例，再返回一个 Looper，避免返回一个空的对象**。

这样说可能不是很清晰，我们结合 IntentService.onCreate 来看：
```java
public void onCreate() {
    super.onCreate();
    HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
    thread.start();

    mServiceLooper = thread.getLooper();
    mServiceHandler = new ServiceHandler(mServiceLooper);
}
```
onCreate 方法中，生成了一个 HandlerThread 的实例，启动线程，然后获取 Looper，因为 HandlerThread 的 Looper 是在 run 方法，即另一个线程中初始化，但是获取是在主线程中获取，因此一定要是有线程同步机制，避免返回 Looper 为空，这就是此处使用 notifyAll/wait 的原因。

## 示例
上面说了这么多，我们来写一个简单的 HandlerThread 来对其进行说明：
```java
HandlerThread mThread = new HandlerThread("DownloadThread");

public void onClick(View view) {
    Log.i("HandlerActivity", "Arirus onClick: ");

    mThread.start();

    Looper looper = mThread.getLooper();

    Handler handler = new Handler(looper, new Handler.Callback() {
        @Override
        public boolean handleMessage(Message msg) {

        Log.i("HandlerActivity", "Arirus handleMessage: start download");

        SystemClock.sleep(5000);

        Log.i("HandlerActivity", "Arirus handleMessage: end download");

        return true;
        }
    });

    handler.sendMessage(Message.obtain());
}
```
出于省事起见，我就直接在 Button 的点击事件中进行线程启动，这样做只能点击一次，不能点击多次，正式使用不能这样。
启动线程后，获取线程的 Looper，创建 Handler 来接受传递的信息，Handler 接受信息后，开始进行耗时的处理。打出的日志是这样的：
```java
09:43:54.520 22825-22825 I/HandlerActivity: Arirus onClick: 
09:43:54.524 22825-22877 I/HandlerActivity: Arirus handleMessage: start download
09:43:59.525 22825-22877 I/HandlerActivity: Arirus handleMessage: end download
```
耗时的操作确实是在子线程中进行的。上面做完了还没有结束，我们知道，HandlerThread 的 run 方法中，调用了 Looper.loop 方法，会一直循环，因此线程才不会退出，我们需要手动退出，所以我们还要在合适的时机调用:

```java
looper.quit();
```
否则线程会一直运行就像这样：

![looper_quit.png](/img/looper_quit.png)


# ActivityThread
最后再来看看 ActivityThread。ActivityThread 是运行在主线程上的一个类，我们在[启动 App](https://arirus.cn/post/android-%E8%B7%A8%E8%BF%9B，%E7%A8%8B%E9%80%9A%E4%BF%A1-%E5%90%AF%E5%8A%A8-app-/)中，对其有过简要的分析，这里我们再来看看其 main 方法：
```java
public static void main(String[] args) {
    ...

    Looper.prepareMainLooper();

    ActivityThread thread = new ActivityThread();
    thread.attach(false);

    if (sMainThreadHandler == null) {
        sMainThreadHandler = thread.getHandler();
    }
    ...
    Looper.loop();

    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```
是不是和 HandlerThread 的 run 方法极其类似，都是先准备 Looper(MessageQueue)，生成 handler(ActivityThread.H)，最后开启 loop。因此从某种意义上来说，主线程也是一个 HandlerThread。

# 小结
本篇，主要是分析了 Handler 模型的三个大类：Handler Looper MessageQueue，Handler 负责对于消息的发送和处理，Looper 负责不断从 MessageQueue 中获取数据，MessageQueue 负责数据的存储和读取。最后结合 Handler 的模型分析了 HandlerThread 和 ActivityThread，那本篇的内容就到此结束，下篇见。