---
date : 2018-11-24T09:52:24+08:00
tags : ["底层", "Android", "跨进程通信"]
title : "Android 跨线程通信 Java 多线程"

---

# 写在最前面

之前我们把 Android 的进程间通信做了介绍，并最后手写实现了跨进程通信。接下来，我们来介绍下 Android 的跨线程通信的相关应用。不同于跨进程的 Binder 一家独大，多线程在语言层面是原生支持的，同时 Android SDK 又封装了跨线程的工具，当然有些第三方库也实现了跨线程实现，因此多线程在 Android 更是一个百花齐放的状态。那本篇先从最基本的开始，即语言层面来说说多线程的相关操作。

<!--more-->

# 线程与进程

这是一个特别特别老生常谈的问题，基本上研究生毕业面试时所有的公司都问过。这里我们通过 [CS：APP](https://book.douban.com/subject/1896753/) 和 [线程与进程](https://developer.android.com/guide/components/processes-and-threads?hl=zh-cn) 两个资料来源来说这个线程与进程是什么区别。

进程是操作系统对一个正在运行的程序的一种抽象。一个系统上可以有多个进程同时运行，每个进程都好像在独自的占用硬件（CUP等）。操作系统保存跟踪进程运行时所需要的所有状态信息，这个状态就是上下文。一个进程是单一的控制流，实际上一个进程可以由多个被称为线程的执行单元组成，每一个线程都运行再进程的上下文中，共享同样的代码和全局数据。

Android 系统中，当某个应用组件启动且该应用没有运行其他任何组件时，Android 系统会使用单个执行线程为应用启动新的 Linux 进程。默认情况下，同一应用的所有组件在相同的进程和线程（称为“主”线程）中运行。 如果某个应用组件启动且该应用已存在进程（因为存在该应用的其他组件），则该组件会在此进程内启动并使用相同的执行线程。

因此可以简单的理解为：Android 中，每一个应用都运行在一个进程中和别的应用完全不互通的，默认情况下所有组件都是在这一个进程当中的，同样默认情况下所有组件都是运行在主线程当中，每一个组件内都可以开启新的线程来执行后台操作。

需要注意的是，我们这里说的线程是操作系统线程，是操作系统利用时间片的方式，把 CPU 的运行拆成多条运行逻辑，单核 CPU 也可以执行多线程操作。而现在的 CPU 大部分都是多核的，例如笔者的计算机 CPU 显示4核4线程，这个线程是 CPU 的物理线程，一个核心对应一个线程。而所谓的超线程技术（Hyper-Threading）就是在硬件级别使得一个核心对应两个物理线程。因此物理线程和操作系统线程是完全不一样的东西。

线程的本质就是按代码顺序执⾏行行下来，执行完毕就结束的一条线。UI 线程处于需要是不能结束的，因为其本身需要实时渲染画面等等操作。所以 UI 线程永远处于一个死循环当中。

# 线程的创建

## Thread 和 Runnable
最常见就是创建一个 Thread ，后执行 start 方法来开启线程的执行（注意不要直接执行 run 方法）。同样还有创建一个 Runnable ，在送到 Thread 中进行执行。这两个方法都是很简单的，我们都不再细说了。当然一个 Thread 可以实现 run 方法，同时将一个 Runnable 传入到 Thread 中，这样两个 run 方法都会执行，不过不要去掉 Thread.run 中的 super 方法。

## ThreadFactory 
ThreadFactory 是一个接口，实例需要重写 newThread 方法，这样每次调用时都会生成一个新的 Thread，其本质和直接 new Thread 没什么区别。
```java
ThreadFactory factory = new ThreadFactory() {
    AtomicInteger count = new AtomicInteger(0);

    @Override
    public Thread newThread(Runnable r) {
        return new Thread(r, "Thread-" + count.incrementAndGet());
    }
};

Runnable runnable = new Runnable() {
    @Override
    public void run() {
        System.out.println(Thread.currentThread().getName() + " started!");
    }
};

Thread thread = factory.newThread(runnable);
thread.start();
```
需要注意的是，ThreadFactory 中如果需要使用 count 作为计数，要用原子数据类进行操作，避免由多线程并发而产生错误。

## Executor 框架与线程池
Executor 框架于 Java 1.5 版本引入，里面包含 Executor，Executors，ExecutorService，线程池，Future，Callable 等众多概念。我们力争通过最少的介绍来将 Executor 框架与线程池的概念建立起来。

### Executor 与 ExecutorService
Executor 是一个接口，也是 Executor 框架最基本的类。接口中之定义了一个方法 execute（Runnable command），该方法接收一个 Runable 实例，它用来执行一个任务，任务即一个实现了 Runnable 接口的类。总体来说 Executor 是线程 Thread 的一个更高级别的抽象，将 Thread 中具体执行的行为抽离出来，所以 Executor 是一种行为的抽象，不管它是建立在线程上还是别的什么上面，只要可以执行某个任务就可以。
```java
class ThreadPerTaskExecutor implements Executor {
    public void execute(Runnable r) {
        new Thread(r).start();
    }
}
```
这个就是通过 Thread 来实现的 Executor。

ExecutorService 接口继承自 Executor 接口，它提供了更丰富的实现多线程的方法。例如：

- void shutdown() //关闭 Executor，不许新的 Runnable 进入队列
- List<Runnable> shutdownNow() //立即关闭 Executor，等待的 Runnable 直接返回
当然除了关闭还有几个执行的方法，这里以 submit(Callable) 举例：
```java
Callable<String> callable = new Callable<String>() {
    @Override
    public String call() {
        try {
            Thread.sleep(1500);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return "Done!";
    }
};

ExecutorService executor = Executors.newCachedThreadPool();
Future<String> future = executor.submit(callable);
try {
    String result = future.get();
    System.out.println("result: " + result);
} catch (InterruptedException | ExecutionException e) {
    e.printStackTrace();
}
```
这里我们用 Executors 的类方法来生成了一个 ExecutorService，生成了一个 Callable，将其传入到 executor.submit 方法中，返回了 Future 对象，注意 Future.get() 方法是阻塞方法，因此前台线程会挂起等待后台线程返回。这种模式当然在 Android 的 UI 线程是不能这么做的。

Callable 和 Future 都是 Executor 框架中的，实现异步分重要类。Callable 类似于 Runnable，两者都是为那些其实例可能被另一个线程执行的类设计的。但是 Runnable 不会返回结果，并且无法抛出经过检查的异常而 Callable 有返回结果，而且当获取返回结果时可能会抛出异常。Callable 中的 call()方法类似 Runnable 的 run()方法，区别同样是有返回值，后者没有。当将一个 Callable 的对象传递给 ExecutorService 的 submit 方法，则该 call 方法自动在一个线程上执行，并且会返回执行结果 Future 对象。

ExecutorService 还有别的一些方法，这里我们就不再演示了，看文档就行。还有一个 ScheduledExecutorService 是继承于 ExecutorService，可以执行周期性任务，方法不复杂，直接看文档就好了。

### Executors 与线程池
Executors 实际上是 Executor 框架的一个工具类和辅助类，常用于创建 ExecutorService，ScheduledExecutorService，ThreadFactory，Callable 的实例。其中 ExecutorService 都是通过创建线程池实例来返回的 ExecutorService 对象。那我们直接看看 ThreadPoolExecutor 的构造函数签名：
```java
public ThreadPoolExecutor(int corePoolSize,
                            int maximumPoolSize,
                            long keepAliveTime,
                            TimeUnit unit,
                            BlockingQueue<Runnable> workQueue,
                            ThreadFactory threadFactory,
                            RejectedExecutionHandler handler)
```
简单说一下各个参数的概念：

- corePoolSize 该线程池中核心线程数最大值，所谓核心线程就是线程池中总数小于核心线程数的线程，这些线程使用完成后不会被删掉，直到线程池的关闭；
- maximumPoolSize 线程池中最大线程个数，线程总数 = 核心线程数 + 非核心线程数。非核心线程使用完成后就会被销毁掉；
- keepAliveTime 非核心线程闲置超时时长，超过这个时间非核心线程就会被销毁掉；
- unit 超时时间单位；
- workQueue 任务队列，存放着待执行的 Runnable 对象。BlockingQueue 是阻塞的，所谓阻塞就是当队列为空时，取元素的操作会等到队列中有元素再返回；也支持添加元素时，如果队列已满，那么等到队列可以放入新元素时再放入。具体方法我们就不看了；
- threadFactory 或许执行的线程；
- handler 异常相关可以忽略。

参数已经很好说明 ThreadPoolExecutor 的概念了，这里我们最常用的几个 Executors 方法：
```java
// 创建具有固定线程数的线程池
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>());
}

//创建有60秒缓冲的线程池
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                    60L, TimeUnit.SECONDS,
                                    new SynchronousQueue<Runnable>());
}

//创建只有一个线程的线程池，所有的 Runnable 只能依次执行    
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}

//创建具有周期性质的线程池，这个用的少
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
    return new ScheduledThreadPoolExecutor(corePoolSize);
}

//ScheduledThreadPoolExecutor():
public ScheduledThreadPoolExecutor(int corePoolSize) {
    super(corePoolSize, Integer.MAX_VALUE,
          DEFAULT_KEEPALIVE_MILLIS, MILLISECONDS,
          new DelayedWorkQueue());
}
```
在不自定义线程池的情况下，以上方法应该是足够用，我们来看一个例子：
```java
Runnable runnable = new Runnable() {
    @Override
    public void run() {
        System.out.println("Thread with Runnable started!");
    }
};

Executor executor = Executors.newCachedThreadPool();
executor.execute(runnable);
executor.execute(runnable);
executor.execute(runnable);

((ExecutorService) executor).shutdown();
```
这里我们创建了一个 CachedThreadPool 让其来执行多个任务，注意当任务执行完成后应该关闭（shut down）线程池，避免资源的一直无法回收。同时在使用线程池时，需要首先判断 ThreadPoolExecutor 是否关闭了，如果关闭后再提交任务会抛出 RejectedExecutionException 异常。

一般来说，CachedTheadPool 在程序执行过程中通常会创建与所需数量相同的线程，然后在它回收旧线程时停止创建新线程，因此它是合理的 Executor 的首选。

# 线程同步

## synchronized
我们知道，线程不同于进程，同一个进程下的不同线程公用内存中内容。多线程同时并发访问的资源叫做临界资源，当多个线程同时访问对象并要求操作相同资源时，分割了原子操作就有可能出现数据的不一致或数据不完整的情况，为避免这种情况的发生，我们会采取同步机制，以确保在某一时刻，方法内只允许有一个线程。java 中通过 synchronized 关键字来实现同步机制叫做互斥锁机制，它所获得的锁叫做互斥锁。

接下来根据几个例子来对 synchronized 的使用进行说明：

### synchronized 修饰方法
```java
private synchronized void count(int newValue) {
    x = newValue;
    y = newValue;
    if (x != y) {
        System.out.println("x: " + x + ", y:" + y);
    }
}
```
synchronized 使用时，运行到此处的线程都会尝试获取一个 monitor，这个 monitor 就相当于一个令牌，持有成功才能运行此处的代码，获取此处的资源。每一个对象实例，都会自动被系统分配一个 monitor ，当 synchronized 修饰一个方法时就是对这个实例的 monitor 进行监视。

当一个运行在此处的线程持有了 monitor，别的线程无法持有，因此如果运行到此处则会阻塞等待，直到重新获取 monitor。而对于没有被 synchronized 修饰的代码，运行到此处的线程都会直接运行。

```java
private synchronized void countXY(int newValue) {
    x = newValue;
    y = newValue;
    if (x != y) {
        System.out.println("x: " + x + ", y:" + y);
    }
}

private synchronized void countAB(int newValue) {
    a = newValue;
    b = newValue;
    if (a != b) {
        System.out.println("a: " + a + ", b:" + b);
    }
}
```

但是，如果一个类中多个属性方法都被 synchronized 修饰，那么所有方法在被运行到时都会检查 monitor 是否持有，无论这个方法中访问资源是否都是同一类需要同步。上例中如果 t1 线程进入到了 countXY 中，这时候 t2 线程试图进行 countAB 就会失败，只有等待 t1 完全释放 monitor 后才能进行操作，这样必然造成了效率的低下。

### synchronized 修饰对象
造成上述的最主要原因是 monitor 不够用了，如果对于 countXY 、countAB 分别有两个 monitor，则可以避免这个问题，实际上  synchronized 也是支持的：
```java
Object monitor1 = new Object();
Object monitor2 = new Object();


private void countXY(int newValue) {
    synchronized(monitor1){
        x = newValue;
        y = newValue;
        if (x != y) {
            System.out.println("x: " + x + ", y:" + y);
        }
    }
}

private void countAB(int newValue) {
    synchronized(monitor2){
        a = newValue;
        b = newValue;
        if (a != b) {
            System.out.println("a: " + a + ", b:" + b);
        }
    }
}
```
这样我们分别使用 monitor1、monitor2 来代替实例 monitor，这时 t1 执行到 countXY 方法时，t2 依然可以继续执行 countAB 方法。

### synchronized 本质

了解 synchronized 本质时，我们需要先了解下 Java 的内存机制。Java 所有变量都存储在主内存中，每个线程都有自己独立的工作内存，里面保存该线程的使用到的变量副本（该副本就是主内存中该变量的一份拷贝）。线程对共享变量的所有操作都必须在自己的工作内存中进行，不能直接在主内存中读写，不同线程之间无法直接访问其他线程工作内存中的变量，线程间变量值的传递需要通过主内存来完成。

因此线程1如果修改共享的变量，想要被线程2及时看到，必须

- 把工作内存1中更新过的共享变量刷新到主内存中
- 将主内存中最新的共享变量的值更新到工作内存2中

因此，对于 synchronized 主要是做了两个工作

- 保证⽅方法内部或代码块内部资源（数据）的互斥访问。即同一时间、由同一个
monitor 监视的代码，最多只能有一个线程在访问，保证资源被唯一操作
- 保证线程之间对监视资源的数据同步。即，任何线程在获取到 monitor 后的第一时
间，会清空自己的工作内存，将共享内存中的数据复制到自己的缓存中；任何线程在释放 monitor 的第一时间前，会先将缓存中的数据复制到共享内存中。

以上就是 synchronized 保证原子性与同步性的原因。

这时候，有同学就要问了，既然 synchronized 这么好，那为啥用起来并不频繁呢？答案的原因在于效率：
```java
private void countXY(int newValue) {
    x = newValue;
    y = newValue;
    if (x != y) {
        System.out.println("x: " + x + ", y:" + y);
    }
}
new Thread() {
    @Override
    public void run() {
        System.out.println("start "+System.currentTimeMillis());
        for (int i = 0; i < 1_000_000_000; i++) {
            count(i);
        }
        System.out.println("end "+System.currentTimeMillis());
    }
}.start();
```
上面是没有添加 synchronized 关键字，我们只开启一个线程，使其运行，有如下日志：
```java
start 1544239185788
end 1544239186153
```
也就是执行 1_000_000_000 次循环，只使用了 365 毫秒，同样的代码如果我们使用 synchronized 修饰日志是：
```java
start 1544239817421
end 1544239839546
```
运行了 1_000_000_000 次循环，使用了 22125 毫秒，是上面的60多倍，原因在于 CPU 在执行代码的时候，为了减少变量访问的时间消耗可能将代码中访问的变量的值缓存到该 CPU 缓存区中，因此，相应的代码再次访问该变量的时候，相应的值可能从 CPU 缓存中而不是主内存中读取的。同样的，代码对这些被缓存过的变量的值的修改也可能仅是被写入 CPU 缓存区，而没有写入主内存。所以效率就是差在这 1_000_000_000 *2 次的与内存交互上。 

## volatile
有一种说法是 volatile 是一种轻量锁，这话对也不对。因为 volatile 关键字具有同步性但不具有原子性，原因在于当使用 volatile 来修饰变量相当于告诉编译器和JVM的内存模型：这个变量是对所有线程共享的、可见的，每次变量写入后都会更新到主内存，同样当变量修改都会从主内存获取而不是 CPU 的缓存。但是 volatile 完全不具有原子性，无法保证一系列的操作不会被分割。当然 volatile 也可以保证指令不被重排序，这部分内容可以看看我在 [Java查缺补漏](/Java查缺补漏.html) 中关于单例模式的说明，这里就不多讲了。

## synchronized 与 volatile
上面用最简单方式说了说这二者的区别下面就简单总结一下：

- synchronized 用于修饰方法和代码块，可以显示定义 monitor，volatile 仅用于修饰变量。
- synchronized 锁定当前代码块，同一个时刻之后有一个线程可以运行在这里，每次运行都会从主内存取资源的拷贝最后再重新写入到主内存，操作具有原子性; volatile 每次运行都会从主内存取资源的拷贝最后再重新写入到主内存，同一时刻可以有多个线程执行到这个变量，操作不具有原子性。
- volatile 不会造成线程的阻塞；synchronized 可能会造成线程的阻塞和上下文切换。
- volatile 不允许进行重排序。
- 使用 synchronized 与 volatile 都会付出性能代价，谨慎使用。

# 死锁
上面说完线程同步，这里我们再来说说死锁。通常死锁是发生在两个 monitor 上，开始当线程 t1 持有 monitor A，线程 t2 持有 monitor B，然后 t1 t2 又想分别持有对方的 monitor，但是都被对方持有，那么双方的等待就陷入了僵局，这就是死锁。

# 线程间通信
上面说的都是两个线程是平行的关系，不存在谁先谁后的关系，但假如有这么一种情况：t1 线程设置变量，t2 线程获取变量，这样 t1 和 t2 之间就会存在明显的先后顺序，即 t1 必须在 t2 开始前结束。但是线程我们只能控制开始时间，无法控制其结束时间，因此进行线程间通信就是十分必要的。这里就要用到 Object.wait() Object.notify() 和 Object.notifyAll() 方法。
```java
private synchronized void initString() {
    sharedString = "rengwuxian";
    notifyAll();
}

private synchronized void printString() {
    while (sharedString == null) {
        try {
            wait();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
    System.out.println("String: " + sharedString);
}

@Override
public void runTest() {

    new Thread(()->{
        try {
            Thread.sleep(500);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        printString();
    }).start();

    new Thread(()->{
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        initString();
    }).start();
}
```
这个例子中，分别起了两个线程，一个做初始化，一个读取，但是读取的线程挂起的时间短，因此会先于初始化的线程，因此，在读取线程在读取字符串时，如果字符串没有初始化就会开始等待，让出 CPU，当初始化线程完成初始化工作，可以进行通知，收到通知后，读取线程会完成剩下的工作。那 notify 和 notifyAll 的区别在哪？以下引用自知乎：

>如果线程调用了对象的 wait()方法，那么线程便会处于该对象的等待池中，等待池中的线程不会去竞争该对象的锁。当有线程调用了对象的 notifyAll()方法（唤醒所有 wait 线程）或 notify()方法（只随机唤醒一个 wait 线程），被唤醒的的线程便会进入该对象的锁池中，锁池中的线程会去竞争该对象锁。也就是说，调用了notify后只要一个线程会由等待池进入锁池，而notifyAll会将该对象等待池内的所有线程移动到锁池中，等待锁竞争优先级高的线程竞争到对象锁的概率大，假若某线程没有竞争到该对象锁，它还会留在锁池中，唯有线程再次调用 wait()方法，它才会重新回到等待池中。而竞争到对象锁的线程则继续往下执行，直到执行完了 synchronized 代码块，它会释放掉该对象锁，这时锁池中的线程会继续竞争该对象锁。


# 小结
本篇中，我们主要是回顾了下 Java 的线程基础，包括线程的创建，线程池的使用，线程间的同步，和线程间的通信。下篇我们还是回归 Android ，来看看 Android 中常用的线程操作都有哪些。