# 四种引用

## 强引用
```java
Object obj = new Object();
```
当JVM的内存空间不足时，宁愿抛出OutOfMemoryError使得程序异常终止也不愿意回收具有强引用的存活着的对象。


## 软引用
```java
Object obj = new Object();
SoftReference softObj = new SoftReference(obj);
obj = null； //去除强引用
```
当JVM认为内存空间不足时，就回去试图回收软引用指向的对象，也就是说在JVM抛出OutOfMemoryError之前，会去清理软引用对象。软引用可以与引用队列(ReferenceQueue)联合使用。

当softObj软引用的obj被GC回收之后，softObj 对象就会被塞到queue中，之后我们可以通过这个队列的poll()来检查你关心的对象是否被回收了，如果队列为空，就返回一个null。反之就返回软引用对象也就是softObj。


## 弱引用
```java
Object obj = new Object();
WeakReference<Object> weakObj = new WeakReference<Object>(obj);
obj = null； //去除强引用
```
在GC的时候，不管内存空间足不足都会回收这个对象，同样也可以配合ReferenceQueue 使用，也同样适用于内存敏感的缓存。ThreadLocal中的key就用到了弱引用。


## 虚引用
```java
Object obj = new Object();
ReferenceQueue queue = new ReferenceQueue();
PhantomReference<Object> phantomObj = new PhantomReference<Object>(obj , queue);
obj = null； //去除强引用
```

虚引用仅仅只是提供了一种确保对象被finalize以后来做某些事情的机制。比如说这个对象被回收之后发一个系统通知啊啥的。虚引用是必须配合ReferenceQueue 使用的。

# Reference 与 ReferenceQueue

## Reference

主要是负责内存的一个状态，当然它还和java虚拟机，垃圾回收器打交道。Reference类首先把内存分为4种状态Active，Pending，Enqueued，Inactive。

- Active，一般来说内存一开始被分配的状态都是 Active，
- Pending 大概是指快要被放进队列的对象，也就是马上要回收的对象，
- Enqueued 就是对象的内存已经被回收了，我们已经把这个对象放入到一个队列中，方便以后我们查询某个对象是否被回收，
- Inactive就是最终的状态，不能再变为其它状态。

## ReferenceQueue

引用队列，在检测到适当的可到达性更改后，垃圾回收器将已注册的引用对象添加到队列中，ReferenceQueue实现了入队（enqueue）和出队（poll），还有remove操作，内部元素head就是泛型的Reference。

采用 Reference + ReferenceQueue 来判断一个对象是否被回收了。

- 创建一个引用队列 queue
- 创建 Refrence 对象，并关联引用队列 queue
- 在 reference 被回收的时候，refrence 会被添加到 queue 中

# WeakHashMap

WeakHashMap不会阻止GC回收key对象（不是value）。其内部有个 ReferenceQueue，每次 put 对象的时候，key 就和次 quene 绑定到一起。

## WeakHashMap如何不阻止对象回收呢

- WeakHashMap的Entry继承了WeakReference
- 其中Key作为了WeakReference指向的对象
- 因此WeakHashMap利用了WeakReference的机制来实现不阻止GC回收Key

WeakHashMap采用的是轮询的形式，在其put/get/size等方法调用的时候都会预先调用一个poll的方法，来检查并删除失效的Entry。

```java
/**
* Expunges stale entries from the table.
*/
private void expungeStaleEntries() {
    for (Object x; (x = queue.poll()) != null; ) {
        synchronized (queue) {
            @SuppressWarnings("unchecked")
                Entry<K,V> e = (Entry<K,V>) x;
            int i = indexFor(e.hash, table.length);

            Entry<K,V> prev = table[i];
            Entry<K,V> p = prev;
            while (p != null) {
                Entry<K,V> next = p.next;
                if (p == e) {
                    if (prev == e)
                        table[i] = next;
                    else
                        prev.next = next;
                    // Must not null out e.next;
                    // stale entries may be in use by a HashIterator
                    e.value = null; // Help GC
                    size--;
                    break;
                }
                prev = p;
                p = next;
            }
        }
    }
}
```

从队列中去除元素，并把相应的结点删除。实际上value的回收是延迟与key的，仅仅是在key被GC后，把value置为null，并不是立即回收。




