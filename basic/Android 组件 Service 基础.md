---
date : 2017-10-12T17:25:45+08:00
tags : ["基础", "Android", "Service"]
title : "Android 组件 Service 基础"

---

# 写在最前面

Service 是 Android 一个可以在后台执行长时间运行操作而不提供用户界面的应用组件。服务可由其他应用组件启动，而且即使用户切换到其他应用，服务仍将在后台继续运行。本篇中我们主要通过[官方文档](https://developer.android.com/guide/components/services)来对 Service 进行一个总结。

<!--more-->

# 服务与线程

服务是一种即使用户未与应用交互也可在后台运行的组件。其和 Activity 的区别跟在于后者使用布局文件，而前者不需要使用。线程则是 CPU 最小调度单位，用来执行某项具体的工作。默认情况下创建服务时，服务还是运行在主线程中的，需要另外单开线程来执行任务。服务是运行在一个进程中的，和线程完全不是一个概念。

那么为什么服务和线程会比较容易搞混呢？以下是个人的一个猜测：我们都知道 Android 系统有5级的进程等级：

    前台进程：包括但不限于 正在交互的 Activity；和其绑定的 Service（onBind调用）；前台 Service 等
    可见进程：包括但不限于 可见的 Activity（悬浮窗后面的 Activity）；和其绑定的 Service
    服务进程：startService() 方法启动的服务且不属于上述两个更高类别进程的进程。
    后台进程：不可见的 Activity。
    空进程：  不含任何活动应用组件的进程。
通常我们使用 Service 是为了**在后台**执行一些耗时的任务，下载，数据库读写，文件 I/O 等。因此，在 8.0 设备之前，部分开发者可能直接使用 StartService 来创建一个服务来执行这些耗时操作，同时入上面说的，直接创建一个 Service 其本身还是运行在 UI 线程，所以需要另起线程来做如上操作。所以可以理解成 Service 和 Thread 是结伴而行，这也是很多人会搞混淆的原因吧。

## 两种启动方式

Service 的使用方法有两种，就像上面说的：第一种是，像启动 Activity 一样使用 StartActivity一样，直接 StartService 来启动一个 Service；另一种是通过 bindService 来于一个 Service绑定进而获取 Service 实例，然后对其进行操作。我们分别来看下两种方式的异同点和注意事项。

### startService

这个一个 Context 方法，就是说不一定要是使用 Activity 来启动，Recivier 也是可以启动的。注意使用 Intent 来启动 Service 时，一定不能用隐式 intent，比较如果有多个 Service 符合条件，无法选择启动哪一个，一定要使用显式的 Intent。可以多次调用 startSevice 方法，不会启动重复的 Service，不像 Activity 可能会出现启动重复的 Activity。每次启动后，都会调用 onStartCommand 方法，无论是不是第一次启动，启动的 Intent 也会传输到这个函数里面。一旦执行此方法，服务即会启动并可在后台无限期运行。如果需要停止需要手动调用 stopSelf() 或 stopService() 来停止服务。官方还给了一个注意事项：如果是要以比较快的频率来执行任务，建议还是使用 bindService，因为这样比较高。

onStartCommand 的返回值通常有 3 种：
  
    START_NOT_STICKY
“非粘性的”。使用这个返回值时，如果在执行完onStartCommand后，服务被异常kill掉，系统不会自动重启该服务

    START_STICKY
如果service进程被kill掉，保留service的状态为开始状态，但不保留递送的intent对象。随后系统会尝试重新创建service，由于服务状态为开始状态，所以创建服务后一定会调用onStartCommand(Intent,int,int)方法。如果在此期间没有任何启动命令被传递到service，那么参数Intent将为null。

    START_REDELIVER_INTENT
重传Intent。使用这个返回值时，如果在执行完onStartCommand后，服务被异常kill掉，系统会自动重启该服务，并将Intent的值传入。

#### stopSelf(startId) 与 stopSelf()

上面说了，我们通过 startService 来启动来一个服务，需要我们手动关闭服务，关闭的两种方式：1.在服务内部调用 stopSelf() 方法或 stopSelf(startId)，其中后者方法主要是针对多次调用 startService ，或者多线程启动服务，当一个线程结束后，服务可能还在别的线程中继续进行。此时可以调用 stop（startId）传入最后一个线程启动时的 startId，这样服务在执行完最后一个线程的工作后才会安全的停止。这里我们给出一个 startService 的样例：

```java
mButton.setOnClickListener(v -> {
    Intent intent = new Intent(this,BackgroundService.class);
    startService(intent);
});

public class BackgroundService extends Service {
  public BackgroundService() {
  }
  ...

  @Override
  public IBinder onBind(Intent intent) {
    return null;
  }

  @Override
  public int onStartCommand(Intent intent, int flags, int startId) {
    Log.i(ARIRUS, "onStartCommand: ");
    Observable.intervalRange(1, 90, 0, 1, TimeUnit.SECONDS)
        .subscribe(aLong -> {
          if (aLong%10==0) stopSelf(startId);
          Log.i(ARIRUS, "onCreate: " + aLong + " " + isMyServiceRunning(BackgroundService.class));
        });
    return Service.START_STICKY;
  }
  ...
}
```

每次点击按钮的时候，启动一个新的线程来进行计数，在最后一次点击10s之后关闭当前的服务，运行多个线程在同时跑这。

上面说了这么多，其实总结起来就是一点**使用 startService 来启动一个服务，这个服务是不受控制的，与启动的组件是没有交互的**。当然我们可以在应用启动的时候传一个 Handler 给服务，但是只能作为服务给 Activity 的通知使用，不能通过 Activity 来操作 Service。那么，如果我们有更为具体复杂的需求，可以考虑使用 bindService 来进行对服务的控制。

### bindService

客户端可通过调用 bindService() 绑定到服务。调用时，它必须提供 ServiceConnection 的实现，后者会监控与服务的连接。bindService() 方法会立即无值返回，但当 Android 系统创建客户端与服务之间的连接时，会对 ServiceConnection 调用 onServiceConnected()，向客户端传递用来与服务通信的 IBinder。

创建提供绑定的服务时，您必须提供 IBinder，用以提供客户端用来与服务进行交互的编程接口。通常可以使用扩展 Binder 类，使用 Messenger，实现 AIDL 来实现。三者的区别用一句话来说明：Binder 用在同一个进程内部服务绑定；Messager 可以用在多进程的服务绑定，不过 Messenger 会在单一线程中创建包含所有请求的队列，因此处理起来就是类似 IntentService ；AIDL 也可以用在多进程的服务绑定，不过不会创建队列来对请求进行同一管理，可以同时对多个请求进行处理。本篇中暂不介绍 AIDL 的相关，只对 Binder 和 Messager 使用进行介绍。

#### Binder

如果服务仅供本地应用使用，不需要跨进程工作，则可以实现自有 Binder 类，客户端通过该类直接访问服务中的公共方法，或者服务本身。
使用步骤：

    1.创建 Binder 实例，包含可以调用的公有方法，或者返回 Service 的实例
    2.从 Service.onBind 回调返回此 Binder 的实例
    3.从 onServiceConnected() 回调方法接收 Binder，并使用提供的方法调用绑定服务

其实就是一点：通过 onServiceConnected 可以获得于 Service 沟通的方式既可，下面给出一个 Demo：

```java
//点击事件来绑定服务 这里会出现重复绑定，只是做代码说明，就不改了，实际不要这么写
mButton.setOnClickListener(v -> {
    Intent intent = new Intent(this,BackgroundService.class);
    bindService(intent, new ServiceConnection() {
    @Override
    public void onServiceConnected(ComponentName name, IBinder service) {
        Log.i(ARIRUS, "onServiceConnected: "+name);
        mBinder = (BackgroundService.ServiceBinder) service;
    }

    @Override
    public void onServiceDisconnected(ComponentName name) {
        Log.i(ARIRUS, "onServiceDisconnected: "+name);
    }
    },BIND_AUTO_CREATE);
});

public class BackgroundService extends Service {
  public BackgroundService() {
  }
  ...

  class ServiceBinder extends Binder {
    final BackgroundService mService;

    ServiceBinder(BackgroundService backgroundService) {
      mService = backgroundService;
    }
  }

  @Override
  public IBinder onBind(Intent intent) {
    return new ServiceBinder(this);
  }
  ...
}
```

服务只用于与其绑定的应用组件，因此如果没有组件绑定到服务，则系统会自动销毁服务，不需要执行 stopSelf 或者 stopService 方法来停止服务。在我们点击 Button 之后，发现log也确实打印出了

    onServiceConnected: ComponentInfo{cn.arirus.versioncomp/cn.arirus.versioncomp.bg.BackgroundService}
同时  onStartCommand 并未执行，因为不是启动一个服务而是绑定服务，所以 onStartCommand 不会执行到。我们需要手动调用 Service 的相关方法。

#### Messenger

当我们的服务处在另一个进程中时，我们可以使用 Messager 来进行进程间通信。类似直接使用 Binder，使用 Messager 也有如下步骤：

    1.服务实现一个 Handler，由其接收来自客户端的每个调用的回调
    2.Handler 用于创建 Messenger 对象（对 Handler 的引用）
    3.Messenger 创建一个 IBinder，服务通过 onBind() 使其返回客户端
    4.客户端使用 IBinder 将 Messenger（引用服务的 Handler）实例化，然后使用后者将 Message 对象发送给服务
    5.服务在其 Handler 中（具体地讲，是在 handleMessage() 方法中）接收每个 Message。
更形象了，相当于直接获取另一个进程中的 Handler，直接向 Handler 中发送 Message 就可以了。

```java
public class BackRemoteService extends Service {
  public BackRemoteService() {
  }

  public static final int REMOTE_SERVICE_HANDLER_TIMER = 307;

  class RemoteServiceHandler extends Handler {
    @Override
    public void handleMessage(Message msg) {
      switch (msg.what) {
        case REMOTE_SERVICE_HANDLER_TIMER:
            Observable.intervalRange(1, 90, 0, 1, TimeUnit.SECONDS).subscribe(aLong -> {
                Log.i(ARIRUS, "onCreate: "+ " " + Thread.currentThread().getName()+" " + aLong + " " + isMyServiceRunning(BackRemoteService.class));
            });
          break;
        default:
          super.handleMessage(msg);
      }
    }
  }

  @Override
  public IBinder onBind(Intent intent) {
    return new Messenger(new RemoteServiceHandler()).getBinder();
  }
}

//调用处
Intent intent = new Intent(this, BackRemoteService.class);
bindService(intent, new ServiceConnection() {
@Override
public void onServiceConnected(ComponentName name, IBinder service) {
    mServerMessenger = new Messenger(service);
    try {
    mServerMessenger.send(
        Message.obtain(null, BackRemoteService.REMOTE_SERVICE_HANDLER_TIMER, 0, 0));
    } catch (RemoteException e) {
    e.printStackTrace();
    }
}

@Override
public void onServiceDisconnected(ComponentName name) {
    mServerMessenger = null;
}
}, BIND_AUTO_CREATE);
```

这样便实现了简单的跨进程通信，实质上就是通过 Handler 来传输一个个的 msg，使用起来也不复杂。

## 返回状态

使用了服务，无论是下载还是执行了服务，我们总是想要知道其工作结果，并根据结果做相应处理或显示在界面上。针对上面的各个情况，我们可以将其分为3类：

    1.startService 和 IntentService 方式，这种方式服务完全是不受控制的，和创建的 Activity 没有联系；
    2.bindService 方式，这种方式 Service 和 Activity 在一个进程内，Activity 可以调用 Service 的公有方法；
    3.Messenger 方法，这种方式 Service 和 Activity 不在同一个进程，需要使用 Handler 进行沟通；

针对上面3类，我们可以使用3种办法进行返回：

    1.发送广播，创建通知。这种是最通用的，无论是那一类都可以使用，注意不要在收到广播时，直接启动Activity。
    2.使用订阅者模式，发布事件。针对上面的一，二类，可以直接使用这个方法。第三类不再一个进程内不适应。
    3.Messenger 自带组件，禁用于上面第三类，IPC的状态返回。
第一种方法最常见，我们就不做单独说明了，我们这里把第二三类方法做个说明。

### 订阅者模式

```java
// 客户端
Intent intent = new Intent(this, BackgroundService.class);
bindService(intent, new ServiceConnection() {
@Override
public void onServiceConnected(ComponentName name, IBinder service) {
    Log.i(ARIRUS, "onServiceConnected: " + name);
    mBinder = (BackgroundService.ServiceBinder) service;
    mCompositeDisposable.add(mBinder.mService.startTimer()
        .observeOn(AndroidSchedulers.mainThread())
        .subscribe(aLong -> mTxt.setText(String.valueOf(aLong))));
}

@Override
public void onServiceDisconnected(ComponentName name) {
    Log.i(ARIRUS, "onServiceDisconnected: " + name);
    mCompositeDisposable.clear();
}
}, BIND_AUTO_CREATE);

// 服务端
Observable<Long> startTimer() {
    return Observable.intervalRange(1, 90, 0, 1, TimeUnit.SECONDS);
}
```

示例如上，绑定后调用 Service 的共用方法，使其返回一个发布者，客户端订阅以后就可以得到所有返回的消息，达到返回状态的效果。当然也可以使用 RxBus 这种事件总线，效果是一样的。需要注意的是如果客户端和服务端不再一个进程中，则此方法完全不适用，切记。

### Messenger 返回状态

这种方式原理也很简单，类似于绑定时，由服务端传给客户端一个 Messenger 性质是一样的。客户端再向服务端发送 Message 时，将自己的 Messenger 一并发回过去，这样服务端就可以根据客户端的 Messenger 来返回数据了。

```java
Handler mHandler = new Handler() {
    @Override
    public void handleMessage(Message msg) {
        switch (msg.what) {
        case CLIENT_HANDLER_WHAT:
            Log.i(ARIRUS, "handleMessage: " + "Got it！");
            break;
        default:
            super.handleMessage(msg);
        }
    }
};

Intent intent = new Intent(this, BackRemoteService.class);
bindService(intent, new ServiceConnection() {
@Override
public void onServiceConnected(ComponentName name, IBinder service) {
    mServerMessenger = new Messenger(service);
    try {
    Message message =
        Message.obtain(null, BackRemoteService.REMOTE_SERVICE_HANDLER_WHAT, 0, 0);
    message.replyTo = new Messenger(mHandler); //将自己的支持的Messenger返回给服务端
    mServerMessenger.send(message);
    } catch (RemoteException e) {
    e.printStackTrace();
    }
}

@Override
public void onServiceDisconnected(ComponentName name) {
    mServerMessenger = null;
}
}, BIND_AUTO_CREATE);


// 服务端接收
  class RemoteServiceHandler extends Handler {
    @Override
    public void handleMessage(Message msg) {
      switch (msg.what) {
        case REMOTE_SERVICE_HANDLER_WHAT:
           Messenger clientMessenger = msg.replyTo;
           clientMessenger.send(Message.obtain(null,CLIENT_HANDLER_WHAT,0,0));
           break;
        default:
          super.handleMessage(msg);
      }
    }
  }

  @Override
  public IBinder onBind(Intent intent) {
    return new Messenger(new RemoteServiceHandler()).getBinder();
  }
```

这里服务端收到 Message 后没有做别的操作，直接将数据返回给了客户端。当客户端返回数据时，将 Message 的 replyTo 字段设置上需要返回的 Messenger 既可。

# 小结

本篇中，我们复习了下服务的最基本用法，两种使用方式：startService 和 bindService，以及二者的异同。接着又对比了使用 bindService 时的两种不同的情况：同一个进程和非同一个进程。两种情况下分别使用 Binder 和 Messenger 来进行客户端与服务器的沟通，最后又分别针对不同方式，总结了使用不同的方式在服务端将状态返回。那本篇的内容就到这里结束了，下篇见。
