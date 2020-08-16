---
date : 2018-12-14T17:18:12+08:00
tags : ["底层", "Android", "跨进程通信"]
title : "Android 跨线程通信 Handler"

---
上一篇，我们了解了 Java 中，我们分析了下 Handler 机制，了解了 Handler Looper MessageQuene 之间的关系，但是对于 Handler 还有一部分要说的。一个消息类型 和 idleHandler。
<!--more-->

# 消息类型

Handler的Message种类分为3种：

- 普通消息
- 屏障消息
- 异步消息

其中普通消息又称为同步消息，屏障消息又称为同步屏障。

我们通常使用的都是普通消息，而屏障消息就是在消息队列中插入一个屏障，在屏障之后的所有普通消息都会被挡着，不能被处理。不过异步消息却例外，屏障不会挡住异步消息，因此可以这样认为：屏障消息就是为了确保异步消息的优先级，设置了屏障后，只能处理其后的异步消息，同步消息会被挡住，除非撤销屏障。

其实对于开发者，异步消息是不暴露使用的。只是对于系统的层面的 API 使用，比较常见的一个例子是 ViewRootImpl.scheduleTraversals中使用了同步屏障

```java
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        //设置同步障碍，确保mTraversalRunnable优先被执行
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        //内部通过Handler发送了一个异步消息
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        if (!mUnbufferedInputDispatch) {
            scheduleConsumeBatchedInput();
        }
        notifyRendererOfFramePending();
        pokeDrawLockIfNeeded();
    }
}
```

通常情况来说，二者使用行为没有差异，只有在设置同步屏障之后，才会有异步消息优先执行的效果。就是简单的设置一个执行的优先级。

## 屏障消息

消息的创建是在 MessageQueue 中，设置实现的：
```java
private int postSyncBarrier(long when) {
    // Enqueue a new sync barrier token.
    // We don't need to wake the queue because the purpose of a barrier is to stall it.
    synchronized (this) {
        final int token = mNextBarrierToken++;
        final Message msg = Message.obtain();
        msg.markInUse();
        msg.when = when;
        msg.arg1 = token;

        Message prev = null;
        Message p = mMessages;
        if (when != 0) {
            while (p != null && p.when <= when) {
                prev = p;
                p = p.next;
            }
        }
        if (prev != null) { // invariant: p == prev.next
            msg.next = p;
            prev.next = msg;
        } else {
            msg.next = p;
            mMessages = msg;
        }
        return token;
    }
}
```
屏障消息没有 target 的设置。这也是为啥上一篇中，我们再说 MessageQuene.next 方法中时，有 target 等于 null 的判断。

而且屏障消息入队是不需要唤醒队列的，因为他其实只是一个过滤作用，不会真正的传递什么 message。

postSyncBarrier返回一个int类型的数值，通过这个数值可以撤销屏障。

```java
Message next() {
    // Return here if the message loop has already quit and been disposed.
    // This can happen if the application tries to restart a looper after quit
    // which is not supported.
    for (;;) {
        ...
        synchronized (this) {
            // Try to retrieve the next message.  Return if found.
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            if (msg != null && msg.target == null) {
                // 如果当前的消息是 消息屏障，那么会返回最近的一个异步消息
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            ...
        }
        ...
    }
}
```

# IdleHandler

```java

通过名字我们可以知道，他其实是一个 “空闲Handler”，既在 Handler 空闲的时候做些什么事情，但是不能保证时机。

public static interface IdleHandler {
    /**
        * Called when the message queue has run out of messages and will now
        * wait for more.  Return true to keep your idle handler active, false
        * to have it removed.  This may be called if there are still messages
        * pending in the queue, but they are all scheduled to be dispatched
        * after the current time.
        */

    boolean queueIdle();
}
```
简单的来说就是，这个 Idle 执行完了之后，要不要从当前的执行队列中清除，返回false就是清除，反之亦然。

之前我们看到 Message.next 方法，里面有如下部分.
```java
Message next() {
    ...
    for (;;) {
       ...

        nativePollOnce(ptr, nextPollTimeoutMillis);

        synchronized (this) {
            // 省略消息队列有数据的情况。
            ...
            // If first time idle, then get the number of idlers to run.
            // Idle handles only run if the queue is empty or if the first message
            // in the queue (possibly a barrier) is due to be handled in the future.
            if (pendingIdleHandlerCount < 0
                    && (mMessages == null || now < mMessages.when)) {
                pendingIdleHandlerCount = mIdleHandlers.size();
            }
            if (pendingIdleHandlerCount <= 0) {
                // No idle handlers to run.  Loop and wait some more.
                mBlocked = true;
                continue;
            }

            if (mPendingIdleHandlers == null) {
                mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
            }
            mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
        }

        // Run the idle handlers.
        // We only ever reach this code block during the first iteration.
        for (int i = 0; i < pendingIdleHandlerCount; i++) {
            final IdleHandler idler = mPendingIdleHandlers[i];
            mPendingIdleHandlers[i] = null; // release the reference to the handler

            boolean keep = false;
            try {
                keep = idler.queueIdle();
            } catch (Throwable t) {
                Log.wtf(TAG, "IdleHandler threw exception", t);
            }

            if (!keep) {
                synchronized (this) {
                    mIdleHandlers.remove(idler);
                }
            }
        }

        // Reset the idle handler count to 0 so we do not run them again.
        pendingIdleHandlerCount = 0;

        // While calling an idle handler, a new message could have been delivered
        // so go back and look again for a pending message without waiting.
        nextPollTimeoutMillis = 0;
    }
}
```

当消息队列有效数据为空的时候，idleHandler 会被调用到，其实也就是在 `nextPollTimeoutMillis = -1;` 赋值的时候，这时候不会有任何的 msg 返回，因此会执行到 mIdleHandlers 的相关代码。

处理完IdleHandler后会将nextPollTimeoutMillis设置为0，也就是不阻塞消息队列，当然要注意这里执行的代码同样不能太耗时，因为它是同步执行的，如果太耗时肯定会影响后面的message执行。

如果一个 IdleHandler 返回了 true 为什么不会造成死循环？

这其实是由于 pendingIdleHandlerCount 的赋值时机，第一进入 next 方法时，会将 pendingIdleHandlerCount 赋值为 -1，如果消息队列为空或者消息执行时间还不到，那么会执行 pendingIdleHandlerCount 初始化的内容。

当执行完一次 pendingIdleHandlerCount 赋值为 0，同时 nextPollTimeoutMillis 赋值为0。同时由于还在for 循环中，尝试获取第二次的队列中的消息，因为 nextPollTimeoutMillis == 0 相当于非挂起等待，直接尝试获取。如果有了Messgae会直接返回，没有的话继续往后执行，但是这一次 pendingIdleHandlerCount == 0，因此会直接开启第三次循环，但这时候 nextPollTimeoutMillis == -1或者下次唤醒间隔。

唤醒之后，必然会有一个相应的 Message 会返回，最终也不再会走到 IdleHandler 的相关内容了。所以这也就是为啥只有第一次空message时会走到 IdleHandler 的处理逻辑，后面都不会了。


