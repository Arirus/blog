---
date : 2018-11-24T09:52:24+08:00
tags : ["底层", "Android", "跨进程通信"]
title : "Android 跨进程通信 手写跨进程 "

---

# 写在最前面

前面几篇中，我们讲解 Android 跨进程的基本概念还通过 App 启动和 Service 绑定来举例说明了 Binder 的使用，其表现就好像是本进程中的一个对象，直接对对象进行操作。我们今天仅根据自己的理解来手写跨进程的实现。

<!--more-->

# 初步实现
本次的模型还是最简单的，创建一个远程服务，使用服务来进行来进行网络请求，简单起见，只是网络请求不做任何的反馈。

## 定义服务
首先，我们先定义一个服务，服务里面包含一个 OkhttpClient 和 Request，方便我们进行请求，除此之外什么都不要。我们可以如下定义：
```java
public class RemoteService extends Service {

  OkHttpClient mClient;
  Request mRequest;

  public RemoteService() {
    mClient = new OkHttpClient.Builder().build();
    mRequest = new Request.Builder().url("http://httpbin.org/get").build();
  }

  @Override public IBinder onBind(Intent intent) {
    return null;
  }
  
  public void request(){
    Response response = null;
    try {
      response = mClient.newCall(mRequest).execute();
      assert response.body() != null;
      Log.i("RemoteService", "Arirus request: "+response.body().string());
    } catch (IOException e) {
      e.printStackTrace();
    }
  }
}
```
request 方法只是进行了一次网络请求，并把结果直接打印出来。这里 IBinder 我们暂时返回 null，等我们下面定义好了再填写。 

## 实现契约

这个契约就是 Binder 本地对象和远程对象同时遵守的，方法同时支持的。所以应该是一个 Interface，同时要继承于 IInterface。IIterface.asBinder 就是 Binder 驱动用于确定这个 Binder 是一个 Binder 对象还是 BinderProxy 对象，进而进行转换。
```java
public interface IContract extends IInterface {
  void request();
}
```
目前我们只需要定义一个 request 方法就可以了。

## 实现 Binder 相关

### Binder 实现类

接下来实现 Binder 也是整个跨进程中最核心的部分了。首先我们要明确几点：

- 这个类必须能和 Service 进行沟通
- 这个类必要要继承于 Binder，即一个本地 Binder
- 这个类如果要实现 IContract 契约
- 这个类要实现 onTransact 方法，接受 BinderProxy 传输过来的参数

那么这里我们可以这样实现：
```java
public class CustomBinder extends Binder implements IContract{

  private RemoteService mService;

  public CustomBinder(RemoteService service) {
    mService = service;
  }

  @Override
  protected boolean onTransact(int code, @NonNull Parcel data, @Nullable Parcel reply, int flags)
      throws RemoteException {
    this.request();
    return true;
  }

  @Override public void request() {
    mService.request();
  }

  @Override public IBinder asBinder() {
    return this;
  }
}
```
由于调用方法不需要任何的参数，因此这里 onTransact 直接调用相应的方法就可以了。

### BinderProxy Helper类
本地 Binder 对象编写完毕，现在考虑 BinderProxy 的相关操作，因为 Client 收到的直接就是 BinderProxy，所以要编写 BinderProxy Helper 来将数据传入到 BinderProxy 中。注意 Helper 并没有真实参与到跨进程的工作中因此，没有实现 IBinder 接口，但是要实现 IContract 接口。
```java
public class CustomBinder extends Binder implements IContract{
  public static class Helper implements IContract{

    IBinder mIBinder;

    public Helper(IBinder iBinder) {
      mIBinder = iBinder;
    }

    @Override public void request() {
      try {
        mIBinder.transact(FIRST_CALL_TRANSACTION,Parcel.obtain(),Parcel.obtain(),0);
      } catch (RemoteException e) {
        e.printStackTrace();
      }
    }

    @Override public IBinder asBinder() {
      return mIBinder;
    }
  }
}
```
Helper 中最关键的就是 request 方法的实现，这里也不需要做什么参数检查和传递，因此只需要使用 IBinder 对象将参数传递给 Binder 驱动就可以了。最后为了保持逻辑上的附属关系，因此将 Helper 类放置于 CustomBinder 中。当然如果像放到外面也是可以的。

### 考虑非跨进程的情况

通常这样写完，我们就可以 bindService ，并且进行对 IBinder 的控制。但是 IBinder 不一定是 BinderProxy 对象，当 Service 和启动的 Activity 在同一个进程时，IBinder 对象实例就是一个 Binder 对象，因此我们应该根据不同的情况，来调用不同对象相应方法。
```java
public class CustomBinder extends Binder implements IContract{
  public static IContract getContract(IBinder binder) throws RemoteException{
    if (binder == null)
      throw new RemoteException("BINDER is null!!");
    if (binder instanceof CustomBinder)
      return (CustomBinder)binder;
    else
      return new Helper(binder);
  }
}
```
在 CustomBinder 实现一个静态方法，当 binder 是一个 CustomBinder 实例此时是没有跨进程的，直接返回他自己就可以了，当 binder 是一个 BinderProxy 实例，这时返回一个 Helper 类实例。

## 调用方法
准备工作都完成了，我们接下就可以直接调用了：
```java
public void onClick(View view){
    Intent intent = new Intent(this,RemoteService.class);
    Log.i("IpcActivity", "Arirus onClick: "+Thread.currentThread().getName());
    bindService(intent, new ServiceConnection() {
        @Override public void onServiceConnected(ComponentName name, IBinder service) {
            try {
                CustomBinder.getContract(service).request();
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }

        @Override public void onServiceDisconnected(ComponentName name) {

        }
    },BIND_AUTO_CREATE);
}
```
我们把日志也打印出来：
```java
进程：cn.arirus.demo
11:19:33.942 I/IpcActivity: Arirus onClick: main
11:19:34.884 I/Choreographer: Skipped 49 frames!  The application may be doing too much work on its main thread.

进程：cn.arirus.demo:remote
11:19:34.883 I/RemoteService: Arirus request: {
      "args": {}, 
      "headers": {
        "Accept-Encoding": "gzip", 
        "Connection": "close", 
        "Host": "httpbin.org", 
        "User-Agent": "okhttp/3.11.0"
      }, 
      "origin": "95.169.2.148", 
      "url": "http://httpbin.org/get"
    }
```
操作没有什么问题，但是主进程中会说主线程操作过多。主要是由于当我们跨进程的时候，C 端的调用线程会挂起等待 S 端的数据返回，因此其实是主进程等待网络进程访问结束，造成的卡顿。如果我们以异步的形式来进行网络请求则不会出现这种情况。所以注意：**客户端调用远程请求时客户端当前的线程会被挂起，直到服务端进程返回数据，所以不能在 UI 线程中发起远程请求**

# 复杂实现
上面的实现方法，完全将一个 API 请求放到了另一个进程中进行，现实中这样的需求几乎没有，都是一些费时费性能的需求才会放到另一个进程中进行操作，那么这里我们来模拟下这个操作：将一个复杂的参数传入到另一个进程中，后者进行一个网络请求，差不多5秒左右才会返回请求结果，将结果返回到主进程中。这里的复杂实现其实就是**将 Parcelable 和 AIDL 类型的数据来进行跨进程传输**。

## 修改契约
首先我们先来修改契约：
```java
public interface IContract extends IInterface {
  void request(RequestParm requestParm , ICallBack callBack);
}

// RequestParm 参数实现
public class RequestParm implements Parcelable {

  String url;
  String parms;

  public RequestParm(String url, String parms) {
    this.url = url;
    this.parms = parms;
  }

  public static final Creator<RequestParm> CREATOR = new Creator<RequestParm>() 
  ...
}

// ICallBack 返回实现
public interface ICallBack extends IInterface {
  void onResponse(String content);
}
```
request 方法不再是无参数的，而是有两个参数，一个参数是 API 的请求参数，需要请求的参数都通过这个来传递给请求进程，另一个参数是 API 的回调参数，通过这个参数做请求进程和主进程的通信。弄清楚了这两个参数的意义，我们想想两个参数分别要这样实现：

- RequestParm 要实现 Parcelable 接口，以满足跨进程传递。
- ICallBack 在两个进程中都有调用，需要实现 IBinder 接口。

## 实现回调对象 Binder
相当于 ICallBack 是另一个跨进程对象（API 响应回调对象）的契约。需要实现相应的 Binder 本地对象类和 Helper 类就好了。
```java
public abstract class ResponseCallBack extends Binder implements ICallBack {

  public static ICallBack getCallBack(IBinder iBinder){
    if (iBinder instanceof ResponseCallBack)
      return (ResponseCallBack)iBinder;
    else
      return new Helper(iBinder);
  }


  @Override public IBinder asBinder() {
    return this;
  }

  @Override
  protected boolean onTransact(int code, @NonNull Parcel data, @Nullable Parcel reply, int flags)
      throws RemoteException {

        
    this.onResponse(data.readString());
    return true;
  }

  public static class Helper implements ICallBack{

    IBinder mIBinder;

    public Helper(IBinder iBinder) {
      mIBinder = iBinder;
    }


    @Override public void onResponse(String content) {
      Parcel data = Parcel.obtain();
      data.writeString(content);
      try {
        mIBinder.transact(FIRST_CALL_TRANSACTION,data,Parcel.obtain(),0);
      } catch (RemoteException e) {
        e.printStackTrace();
      }
    }

    @Override public IBinder asBinder() {
      return mIBinder;
    }
  }
}
```
这里 ResponseCallBack 定义为了一个抽象类，在实现的时候需要实现 onResponse 方法就能接受 API 请求进程返回的数据了。

## 修改 CustomBinder 方法

同样我们需要修改 CustomBinder 的方法，因为契约 IContract 中方法签名修改过了：
```java
public class CustomBinder extends Binder implements IContract{
  ...

  @Override
  protected boolean onTransact(int code, @NonNull Parcel data, @Nullable Parcel reply, int flags)
      throws RemoteException {
    RequestParm requestParm = data.readParcelable(RequestParm.class.getClassLoader());
    ICallBack callBack = ResponseCallBack.getCallBack(data.readStrongBinder());
    this.request(requestParm,callBack);
    return true;
  }

  @Override public void request(RequestParm requestParm, ICallBack callBack) {
    mService.request(requestParm, callBack);
  }

  public static class Helper implements IContract{
    ...

    @Override public void request(RequestParm requestParm, ICallBack callBack) {
      try {
        Parcel request = Parcel.obtain();
        request.writeParcelable(requestParm,0);
        request.writeStrongBinder(callBack.asBinder());
        mIBinder.transact(FIRST_CALL_TRANSACTION,request,Parcel.obtain(),0);
      } catch (RemoteException e) {
        e.printStackTrace();
      }
    }

  }
}
```
这里有两点需要注意：

- Parcel 在写入或者读取 Parcelable 数据时，直接调用 writeParcelable/readParcelable 方法即可，其内部实际上调用的就是 Parcelable.writeToParcel 和 creator.createFromParcel 方法就是我们在 Parcelable 实现时定义的方法。
- Parcel 读取的 Binder 类型数据需要通过调用 ResponseCallBack.getCallBack 方法来将 IBinder 转换成 ICallBack ，原因还是要考虑非跨进程的情况。

## 修改 Service 并调用

然后就是修改 Service 的 request 方法，使其接受参数并将请求返回给主进程：
```java
  void request(RequestParm requestParm , ICallBack callBack){
    mRequest = mRequest.newBuilder().url(requestParm.url)
        .post(new FormBody.Builder().add("args",requestParm.parms).build())
        .build();

    mClient.newCall(mRequest).enqueue(new Callback() {
      @Override public void onFailure(Call call, IOException e) {
        callBack.onResponse(e.getMessage());
      }

      @Override public void onResponse(Call call, Response response) throws IOException {
        callBack.onResponse(response.body().string());
      }
    });
  }
```
最后我们在主线程端就可以直接调用了：
```java
bindService(intent, new ServiceConnection() {
  @Override public void onServiceConnected(ComponentName name, IBinder service) {
    try {
      CustomBinder.getContract(service).request(
          new RequestParm("http://httpbin.org/post", "NMSL"), new ResponseCallBack() {
            @Override public void onResponse(String content) {
              Log.i("IpcActivity", "Arirus onResponse: "+content);
            }
          });
    } catch (RemoteException e) {
      e.printStackTrace();
    }
  }

  @Override public void onServiceDisconnected(ComponentName name) {

  }
},BIND_AUTO_CREATE);
```
调用的结果和我们之前的大致一致，因为这里传递了参数，所以使用了 Post 方法，性质都是一样的。其实这里主要是理清两点就是 RequestParm 与 ResponseCallBack 的定位问题。前者就是一个简单的传递参数，实现 Parcelable 接口就好了，后者是实实在在需要在两个进程间进行操作，因此需要 IBinder 接口的实现。

# CallBack 存储

目前我们实现的 C/S 模型是一对一的，即客户端驱动服务端去做某件事，然后收到服务端的回调，现在如果是多对一的模型即多个客户端来监听服务端的服务，那么就需要考虑 CallBack 的存储了，因为这个过程必然会设计到监听器的 register 与 unregister。如果单纯的根据 ResponseCallBack 对象来进行注册和反注册一定是有问题，因为我们通过第一篇知道所谓数据跨进程不过是一次序列化和反序列化的过程，因此同一个对象的多次跨进程，得到的对象一定不一样：
```java
//第一次跨进程
remote I/RemoteService: Arirus request: cn.arirus.mddemo.ipc.ResponseCallBack$Helper@14a36acf
remote I/RemoteService: Arirus request: android.os.BinderProxy@593cf2e
//第二次跨进程
remote I/RemoteService: Arirus request: cn.arirus.mddemo.ipc.ResponseCallBack$Helper@3f39bc5c
remote I/RemoteService: Arirus request: android.os.BinderProxy@593cf2e
//第三次跨进程
remote I/RemoteService: Arirus request: cn.arirus.mddemo.ipc.ResponseCallBack$Helper@e216b65
remote I/RemoteService: Arirus request: android.os.BinderProxy@593cf2e
```
上面是同一个对象在三次跨进程时对象的数据，可以看到每次得到的 ICallBack 对象都不一样，但是调用其 asBinder 方法，得到的 BinderProxy 对象都是同一个对象，因此我们可以使用 BinderProxy 来做对象一致性的检查，统一管理。RemoteCallbackList 就是这样的一个 CallBack 管理类，可以直接使用就好了，具体使用比较简单，这里就不做过多说明，看文档就行。

# 小结
本篇试图在自己理解 Binder 的基础上来进行手写 Binder 本地类和 Helper 类来进行跨进程传输。同时，又分别实现了 Parcelable 对象和 IInterface 接口，来实现跨进程回调和传输数据。最后真的多对一的 C/S 模型，分析为什么不能注册/反注册监听器。那么本篇的内容大致就是这些，下一篇再见。
