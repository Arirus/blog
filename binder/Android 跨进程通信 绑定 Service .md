---
date : 2018-11-21T14:12:01+08:00
tags : ["底层", "Android", "跨进程通信"]
title : "Android 跨进程通信 绑定 Service "

---

# 写在最前面

上一篇中，我们分析了从 Launcher 启动一个 App 整体的流程。本篇我们就仿照上篇一篇的分析方法来分析 Service 绑定是怎样的一个过程。本篇采用的源码版本是 API 27。

![跨进程绑定服务](/img/BindService.png)

<!--more-->

# 向 ActivityManagerService 发出请求 
我们直接在 Activity 中调用 bindService 方法。由于 Activity 继承于 ContextImpl，因此直接调用了 ContextImpl.bindServiceCommon 方法:
```java
private boolean bindServiceCommon(Intent service, ServiceConnection conn, int flags, Handler handler, UserHandle user) {
    IServiceConnection sd;
    sd = mPackageInfo.getServiceDispatcher(conn, getOuterContext(), handler, flags);
    ...
    int res = ActivityManager.getService().bindService(
    mMainThread.getApplicationThread(), getActivityToken(), service,
    service.resolveTypeIfNeeded(getContentResolver()),
    sd, flags, getOpPackageName(), user.getIdentifier());
    ...
}
```
将 ServiceConnection 转换成 IServiceConnection, 并直接跨进程传递给了 ActivityManagerService.bindService 来进行绑定操做。

# ActivityManagerService 创建进程 
ActivityManagerService 中经过调用转到 ActiveServices.bindServiceLocked 方法：
```java
int bindServiceLocked(IApplicationThread caller, IBinder token, Intent service,
String resolvedType, final IServiceConnection connection, int flags,
String callingPackage, final int userId){
   ...
    if ((flags&Context.BIND_AUTO_CREATE) != 0) {
        s.lastActivity = SystemClock.uptimeMillis();
        if (bringUpServiceLocked(s, service.getFlags(), callerFg, false,
                permissionsReviewRequired) != null) {
            return 0;
        }
    }
    ... 
    requestServiceBindingLocked(s, b.intent, callerFg, false);
    ...
}
```
当我们传入的 flag 包含 BIND_AUTO_CREATE，则会先判断此 Service 是否已经创建成功。
```java
private String bringUpServiceLocked(ServiceRecord r, int intentFlags, boolean execInFg,
boolean whileRestarting, boolean permissionsReviewRequired){
    ...
    app = mAm.getProcessRecordLocked(procName, r.appInfo.uid, false);
    if (app != null && app.thread != null) {
        ...
        realStartServiceLocked(r, app, execInFg);
        return null;
    }
    ...
    if (app == null && !permissionsReviewRequired) {
        mAm.startProcessLocked(procName, r.appInfo, true, intentFlags,
                    hostingType, r.name, false, isolated, false);
        ...
    }
}
```
这里会判断是否存在所需的进程，如果存在直接 realStartServiceLocked 调用会调用 app.thread.scheduleCreateService 方法，类似上一篇的 app.thread.scheduleCreateActivity 就是直接创建了 Service，如果进程不存在，则要先通过调用 startProcessLocked 创建新的进程。就跟上一篇的思路一样的。我们这里还以第二种情况为例，假如没有目标进程，则会先创建进程，创建新的 ActivityThread，将 ApplicationThread 和 AMS 向绑定：
```java
private final boolean attachApplicationLocked(IApplicationThread thread,
int pid) {
    ProcessRecord app;
    ...
    app.makeActive(thread, mProcessStats);
    ...
    thread.bindApplication(processName, appInfo, providers,
        app.instr.mClass,
        profilerInfo, app.instr.mArguments,
        app.instr.mWatcher,
        app.instr.mUiAutomationConnection, testMode,
        mBinderTransactionTrackingEnabled, enableTrackAllocation,
        isRestrictedBackupMode || !normalMode, app.persistent,
        new Configuration(getGlobalConfiguration()), app.compat,
        getCommonServicesLocked(app.isolated),
        mCoreSettingsObserver.getCoreSettingsLocked(),
        buildSerial);
    ...
    if (mStackSupervisor.attachApplicationLocked(app)) {
        didSomething = true;
    }
    ...
    // Find any services that should be running in this process...
    if (!badApp) {
        try {
            didSomething |= mServices.attachApplicationLocked(app, processName);
            checkTime(startTime, "attachApplicationLocked: after mServices.attachApplicationLocked");
        } catch (Exception e) {
            Slog.wtf(TAG, "Exception thrown starting services in " + app, e);
            badApp = true;
        }
    }
}
```
AMS.attachApplicationLocked 这个方法，我们上篇已经分析过了，这次创建新的进程又一次调用到了这个方法。这次没有 Activity 需要添加，只有一个 Service 需要操作，便调用了 ActiveServices.attachApplicationLocked 方法，将 Services 和 Application 向关联。
```java
boolean attachApplicationLocked(ProcessRecord proc, String processName){
    if (mPendingServices.size() > 0) {
        ServiceRecord sr = null;
        for (int i=0; i<mPendingServices.size(); i++) {
            sr = mPendingServices.get(i);
            ...
            realStartServiceLocked(sr, proc, sr.createdFromFg);
            ...
        }
    }
}
```
这样调用，便像 bringUpServiceLocked 方法一样调用了 realStartServiceLocked 方法来启动 Service。

# ApplicationThread 创建服务
```java
private final void realStartServiceLocked(ServiceRecord r,
ProcessRecord app, boolean execInFg){
    ...
    app.thread.scheduleCreateService(r, r.serviceInfo,
        mAm.compatibilityInfoForPackageLocked(r.serviceInfo.applicationInfo),
        app.repProcState);
    ...
    requestServiceBindingsLocked(r, execInFg);
}
```
这样便将创建 Service 的任务发送给了 ApplicationThread 发送了过去。最后都是 ApplicationThread 通知 ActivityThread 来做事情
```java
private void handleCreateService(CreateServiceData data) {
    ...
    java.lang.ClassLoader cl = packageInfo.getClassLoader();
    service = (Service) cl.loadClass(data.info.name).newInstance();
    ...
    ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
    context.setOuterContext(service);

    Application app = packageInfo.makeApplication(false, mInstrumentation);
    service.attach(context, this, data.info.name, data.token, app,
            ActivityManager.getService());
    service.onCreate();
    mServices.put(data.token, service);
    ActivityManager.getService().serviceDoneExecuting(data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
}
```
代码很清晰了，创建 Service 和 Context 绑定，同时也说明运行的线程就是主线程，将 Service 保存到 HashMap 中，最后通知 AMS 完成了创建。由于跨进程是同步进行的，当 ActivityThread 创建完成了 Service 之后，realStartServiceLocked 方法中的 requestServiceBindingsLocked 才会被调用到。那这里我们再看看绑定这里是怎么做的。

# ApplicationThread 绑定服务
ActivitServices 中直接调用 ServiceRecord 进程所在的 Applications 的 scheduleBindService方法：
```java
private final boolean requestServiceBindingLocked(ServiceRecord r, IntentBindRecord i,            boolean execInFg, boolean rebind){
    ...
    r.app.thread.scheduleBindService(r, i.intent.getIntent(), rebind, r.app.repProcState);
    ...
}
```
当然最后还是再 ActivityThread 中进行绑定：
```java
private void handleBindService(BindServiceData data) {
    Service s = mServices.get(data.token);
    ...
    if (s != null) {
        ...
        if (!data.rebind) {
            IBinder binder = s.onBind(data.intent);
            ActivityManager.getService().publishService(
                    data.token, data.intent, binder);
        } else {
            s.onRebind(data.intent);
            ActivityManager.getService().serviceDoneExecuting(
                    data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
        }
        ensureJitEnabled();
        ...
    }
}
```
这样就把 Service 和调用者的 Intent 绑定了起来。最后还将 onBind 返回的 IBinder 返回给了 AMS，进而调用了 ActivitServices.publishServiceLocked 。最后我们看下 ActivitServices 是怎么处理这个的：
```java
void publishServiceLocked(ServiceRecord r, Intent intent, IBinder service) {
    ...
    for (int conni=r.connections.size()-1; conni>=0; conni--) {
        ArrayList<ConnectionRecord> clist = r.connections.valueAt(conni);
        for (int i=0; i<clist.size(); i++) {
            ConnectionRecord c = clist.get(i);
            ...
            try {
                c.conn.connected(r.name, service, false);
            } catch (Exception e) {
               ...
            }
        }
    }
    ...
}
```
最后其实调用的是：ConnectionRecord.IServiceConnection.connected 方法，而 IServiceConnection 又是 ServiceConnection 转换来的，所以是 ServiceConnection 与 IBinder 建立了联系，至此跨进程绑定一个 Service 就此结束了。

# 小结
整体来说，会了上一篇的分析再看绑定 Service 的操作，感觉会清晰不少，毕竟也是由于不需要操作 Activity 整体来说代码思路会更加清晰一些。本篇的内容到此结束，下篇再见。