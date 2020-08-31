 # SharedPreferences

 # 阻塞主线程

SP 是会阻塞主线程的，我们就来看看，其第一次初始化到读取的过程：

## 创建
```java
//ContextImpl.getSharedPreferences
public SharedPreferences getSharedPreferences(File file, int mode) {
    SharedPreferencesImpl sp;
    synchronized (ContextImpl.class) {
        final ArrayMap<File, SharedPreferencesImpl> cache = getSharedPreferencesCacheLocked();
        sp = cache.get(file);
        if (sp == null) {
            checkMode(mode);
            ...
            // 构造真正的实现类
            sp = new SharedPreferencesImpl(file, mode);
            cache.put(file, sp);
            return sp;
        }
    }
    ...
    return sp;
}

// 第一次创建的时候会创建新的缓存对象
private ArrayMap<File, SharedPreferencesImpl> getSharedPreferencesCacheLocked() {
    if (sSharedPrefsCache == null) {
        sSharedPrefsCache = new ArrayMap<>();
    }

    final String packageName = getPackageName();
    ArrayMap<File, SharedPreferencesImpl> packagePrefs = sSharedPrefsCache.get(packageName);
    if (packagePrefs == null) {
        packagePrefs = new ArrayMap<>();
        sSharedPrefsCache.put(packageName, packagePrefs);
    }

    return packagePrefs;
}

// SharedPreferencesImpl 真正的实现类
SharedPreferencesImpl(File file, int mode) {
    mFile = file;       // 对应文件
    mMode = mode;       
    mLoaded = false;    // 是否已经读取成功
    mMap = null;        // 文件解析的内容
    mThrowable = null;
    startLoadFromDisk();// 真正开始读取
}

// 创建一个线程来进行从磁盘中装载
private void loadFromDisk() {
    synchronized (mLock) {
        if (mLoaded) {
            return;
        }
        ...
    }

    // 读取xml文件，并且把他映射成map文件
    Map<String, Object> map = null;
    str = new BufferedInputStream(new FileInputStream(mFile), 16 * 1024);
    map = (Map<String, Object>) XmlUtils.readMapXml(str);
    
    // 赋值给 mMap
    synchronized (mLock) {
        mLoaded = true;
        mThrowable = thrown;

        mMap = map;
        mStatTimestamp = stat.st_mtim;
        mStatSize = stat.st_size;
    } catch (Throwable t) {
        mThrowable = t;
    } finally {
        mLock.notifyAll();
    }
}
```
可以看到创建对象的时候，是新起线程来进行创建，最终返回这个对象，这个过程其实是非阻塞主线程的，但是子线程在启动之后有相应的io操作，因此还是比较耗时间的。

## 读写操作
这里我们直接来进行读写操作。可能我们在平常最大就是初始化之后就进行相应的读写操作。

### 读操作
```java
@Override
public int getInt(String key, int defValue) {
    synchronized (mLock) {
        awaitLoadedLocked();
        Integer v = (Integer)mMap.get(key);
        return v != null ? v : defValue;
    }
}

private void awaitLoadedLocked() {
    // 如果没有载入成功，那么就死循环等待 lock 被唤醒
    while (!mLoaded) {
        try {
            mLock.wait();
        } catch (InterruptedException unused) {
        }
    }
    if (mThrowable != null) {
        throw new IllegalStateException(mThrowable);
    }
}
```
读操作其实没有什么好说的，同时其说明了读操作是线程安全的。从内存缓存中直接读取相应的 v 来直接返回。

### 写操作
SP 的写操作，我们都知道要用 Editor 来进行操作的。
```java
// 如果只是写操作也是需要等完全载入到内存当中
// SharedPreferencesImpl.edit()
public Editor edit() {
    synchronized (mLock) {
        awaitLoadedLocked();
    }
    return new EditorImpl();
}


// EditorImpl.putString()
@Override
public Editor putString(String key, @Nullable String value) {
    synchronized (mEditorLock) {
        mModified.put(key, value);
        return this;
    }
}
```
这里直接把写入的操作放入到 mModified 中，此时也不会直接提交

#### Commit
提交有两种，一个是同步的，一个是异步的。我们先看同步的
```java
 public boolean commit() {

    // 提交到内存当中，获取最新的结果
    MemoryCommitResult mcr = commitToMemory();

    // 执行写操作
    SharedPreferencesImpl.this.enqueueDiskWrite( mcr, null );
    try {
        // 中断等待，当 CountDownLatch == 0 时才会执行，可以被打断
        mcr.writtenToDiskLatch.await();
    } catch (InterruptedException e) {
        return false;
    }
    // 通知相应的 Listener
    notifyListeners(mcr);
    // 返回写入结果
    return mcr.writeToDiskResult;
}

// 从 mMap 读取内容，设置副本，更新 mModify 里的内容到 mapToWriteToDisk，并生成相应的结果。
private MemoryCommitResult commitToMemory() {
    long memoryStateGeneration;
    List<String> keysModified = null;
    Set<OnSharedPreferenceChangeListener> listeners = null;
    Map<String, Object> mapToWriteToDisk;

    synchronized (SharedPreferencesImpl.this.mLock) {
        mMap = new HashMap<String, Object>(mMap);
        
        mapToWriteToDisk = mMap;

        synchronized (mEditorLock) {
            boolean changesMade = false;

            if (mClear) {
                if (!mapToWriteToDisk.isEmpty()) {
                    changesMade = true;
                    mapToWriteToDisk.clear();
                }
                mClear = false;
            }

            for (Map.Entry<String, Object> e : mModified.entrySet()) {
                String k = e.getKey();
                Object v = e.getValue();
                if (v == this || v == null) {
                    if (!mapToWriteToDisk.containsKey(k)) {
                        continue;
                    }
                    mapToWriteToDisk.remove(k);
                } else {
                    if (mapToWriteToDisk.containsKey(k)) {
                        Object existingValue = mapToWriteToDisk.get(k);
                        if (existingValue != null && existingValue.equals(v)) {
                            continue;
                        }
                    }
                    mapToWriteToDisk.put(k, v);
                }

                changesMade = true;
                if (hasListeners) {
                    keysModified.add(k);
                }
            }
            mModified.clear();

        }
    }
    return new MemoryCommitResult(memoryStateGeneration, keysModified, listeners,
            mapToWriteToDisk);
}

private void enqueueDiskWrite(final MemoryCommitResult mcr, null) {
    final boolean isFromSyncCommit = true;

    final Runnable writeToDiskRunnable = new Runnable() {
            @Override
            public void run() {
                synchronized (mWritingToDiskLock) {
                    // 直接写入到文件。
                    writeToFile(mcr, isFromSyncCommit);
                }
            }
        };

    if (isFromSyncCommit) {
        writeToDiskRunnable.run();
        return;
    }
}

// 写入成功后调用 writtenToDiskLatch.countDown();
// 这样之前的 等待的部分放开。最后返回相应的结果。
private void writeToFile(MemoryCommitResult mcr, boolean isFromSyncCommit) {
    ...
    mcr.setDiskWriteResult(true, true);
    ...
}
```

OK，commit 的过程就是：创建 Editor -> 更新 Modify -> 更新到内存中 -> 等待 -> 更新到本地文件 -> 等待结束 -> 返回操作结果

#### Apply
接下来我们看看 异步操作
```java
public void apply() {
    // 提交到内存当中，获取最新的结果
    // 跟commit相同不再分析了
    final MemoryCommitResult mcr = commitToMemory();

    // 等待提交
    final Runnable awaitCommit = new Runnable() {
            @Override
            public void run() {
                try {
                    mcr.writtenToDiskLatch.await();
                } catch (InterruptedException ignored) {
                }
            }
        };

    // 添加到队列中
    QueuedWork.addFinisher(awaitCommit);

    Runnable postWriteRunnable = new Runnable() {
            @Override
            public void run() {
                awaitCommit.run();
                QueuedWork.removeFinisher(awaitCommit);
            }
        };
    
    // 增加到写队列
    SharedPreferencesImpl.this.enqueueDiskWrite(mcr, postWriteRunnable);

    // listener 回调
    notifyListeners(mcr);
}

// 尝试写入
private void enqueueDiskWrite(final MemoryCommitResult mcr,
                                final Runnable postWriteRunnable) {
    final boolean isFromSyncCommit = false;

    final Runnable writeToDiskRunnable = new Runnable() {
            @Override
            public void run() {
                synchronized (mWritingToDiskLock) {
                    writeToFile(mcr, false);
                }

                postWriteRunnable.run();
            }
        };

    // 将准备写入的Runnable 传入到 QuenedWork 中
    QueuedWork.queue(writeToDiskRunnable, true);
}

// QueuedWork.queue
public static void queue(Runnable work, boolean shouldDelay) {

    // 创建一个线程，绑定一个 Handler。
    HandlerThread handlerThread = new HandlerThread("queued-work-looper",
            Process.THREAD_PRIORITY_FOREGROUND);
    handlerThread.start();

    Handler handler  = new QueuedWorkHandler(handlerThread.getLooper());

    synchronized (sLock) {
        sWork.add(work);

        if (shouldDelay && sCanDelay) {
            handler.sendEmptyMessageDelayed(QueuedWorkHandler.MSG_RUN, DELAY);
        } else {
            handler.sendEmptyMessage(QueuedWorkHandler.MSG_RUN);
        }
    }
    // 这都是老套路了。新起线程做操作。最后在新线程的 HandlerThread 中执行 work 的内容。
    // work 就是我们上一步传入的  writeToDiskRunnable。这时会写入到本地文件。结束之后会把 
    // awaitCommit从队列中移除。
}
```
其流程为： 创建 Editor -> 更新 Modify -> 更新到内存中 -> 在QueuedWork入队 -> 创建 HandlerThread -> 更新到本地文件 -> 在QueuedWork出队

但是这么看，好像 writtenToDiskLatch.await() 没有用呀，因为他是在更完本地文件之后才调用的。 这里就是阻塞主线程，爆出 ANR 的关键。

## 导致 ANR
我们知道，在 内存 写入到 本地文件时，往 QueuedWork 增加了一个 awaitCommit，他是要保证写入的内容不会丢失，所以才会将每个apply的await存起来，然后依次调用，如果有没有完成的，则阻塞调用者也就是主线程。

在 ActivityThread 相关方法中有 相关调用，例如handleStopActivity()
```java
public void handleStopActivity(IBinder token, boolean show, int configChanges,
            PendingTransactionActions pendingActions, boolean finalStateRequest, String reason) {
    ...
    QueuedWork.waitToFinish();
    ...
}

public static void waitToFinish() {
    ...
     while (true) {
        Runnable finisher;

        synchronized (sLock) {
            finisher = sFinishers.poll();
        }

        if (finisher == null) {
            break;
        }

        finisher.run();
    }
    ...
}
```

这里调用的就是刚才的 awaitCommit， 会每次都会阻塞的调用。如果提交的多了，那么会出现 主线程的阻塞情况。

## 进程间通信
SP 不支持多进程访问。
- 不同apk(不同packageName,且不具有相同uid)的多进程，MODE_WORLD_READABLE或MODE_WORLD_WRITABLE 对所有人可见，无法针对某一个App，同时 android N上对targetsdk >= 24的应用已明确禁止这两个mode，因此不再讨论。
- 同一个packageName或具有相同uid的package里面存在多个进程：
    MODE_MULTI_PROCESS的作用就是在每次获取SharedPreferences实例的时候尝试从磁盘中加载修改过的数据，并且读取是在异步线程中，只要读的时候另一进程在写，就会导致写丢失，因此一个线程的修改最终会反映到另一个线程，但不能立即反映到另一个进程，所以通过SharedPreferences无法实现多进程同步。
    综合: 如果仅仅让多进程可访问同一个SharedPref文件，不需要设置MODE_MULTI_PROCESS, 如果需要实现多进程同步，必须设置这个参数，但也只能实现最终一致，无法即时同步。

### 为什么不做写失败重试
写文件的步骤：
- 将原文件重命名为mBackupFile
- 重新创建原文件mFile, 并将内容写入其中
- 删除mBackupFile

写进程并不能发现写失败的情况。

# MMKV 原理

## 内存映射
mmap（memory mapping），将文件映射到一块内存区域中，在内存中进行写数据，会由系统直接写到文件当中。由于是系统内核上的操作，因此会直接反映到文件上，不会担心由于crash导致丢失数据。

## 写入优化

使用 protobuf 来写入，每次写入的时候，都是在内存尾部 append 数据。更新的时候由后面的值来覆盖前面的值。

## 空间增长

以内存 pagesize 为单位申请空间，在空间用尽之前都是 append 模式；当 append 到文件末尾时，进行文件重整、key 排重，尝试序列化保存排重结果；排重后空间还是不够用的话，将文件扩大一倍，直到空间足够。

## 多进程访问

 - 使用 ContentProvider。Binder 的 CS 模型慢，因为还要通过 Binder 的转换，使用 BinderProxy 操作进行会有较大的延迟。
 - 增加进程锁
    - FileLock：基于文件描述符的文件锁是天然 robust，缺点是不支持递归加锁，也不支持读写锁升级/降级，需要自行实现

### 进程同步
- 状态同步
    - 写指针的同步：把最新的写指针位置也写到 mmap 内存中
    - 内存重整的感知：使用一个单调递增的序列号
    - 内存增长的感知：通过查询文件大小来获得

```java
void checkLoadData() {
    if (m_sequence != mmapSequence()) {
        m_sequence = mmapSequence();
        if (m_size != fileSize()) {
            m_size = fileSize();
            // 处理内存增长
        } else {
            // 处理内存重整
        }
    } else if (m_actualSize != mmapActualSize()) {
        auto lastPosition = m_actualSize;
        m_actualSize = mmapActualSize();
        // 处理写指针增长
    } else {
        // 什么也没发生
        return;
    }
}
```
- 文件锁采用增加读写锁计数器方式来进行修改。


# SP 多进程使用 ContentProvider


