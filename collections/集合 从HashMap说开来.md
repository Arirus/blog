# 从HashMap说开来

## ConcurrentHashMap

整体思路是类似 HashMap 的，不过使用 CAS 和 synchronized 联合来保证线程安全。

CAS 操作：
```
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
    return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
}

static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                    Node<K,V> c, Node<K,V> v) {
    return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
}

static final <K,V> void setTabAt(Node<K,V>[] tab, int i, Node<K,V> v) {
    U.putObjectVolatile(tab, ((long)i << ASHIFT) + ABASE, v);
}
```

不像 HashTable 对整个方法进行拿锁，以put方法为例：
当且仅当在往bucket中添加非第一个数据的时候，才会使用 synchronized 来进行拿锁。

其余情况使用 CAS 操作来保证线程安全的。CAS指令执行时，当且仅当V符合A时，处理器才会用B更新V的值，否则它就不执行更新。但是，不管是否更新了V的值，都会返回V的旧值。


## HashTable

整体思路类似 HashMap 1.7 版本。与1.8 版本 HashMap 的区别：
- 没有使用红黑树进行数据处理
- 线程安全，不过是使用 synchronized 来修饰方法调用来保证
- 没有使用 二次 hash 来保证，高位也会参与到运算当中
- 没有使用 & 的操作来获取 index，而还是 % 取余的方式
- 扩容的时候
    - 初始容量变成11，而不再是2的n此幂
    - 扩容的时候，容量翻倍+1
- 插入数据的时候，使用头插法，而非尾插法。

## HashSet
HashSet 内部包含一个 HashMap，其实现也是判断key是否多次出现，本质上用的就是 HashMap，value 都是一个固定的 Object 对象。

## LinkedHashMap

继承于 HashMap ，基本上所有的关键方法都是用的 HashMap 来进行实现的，仅有部分方法是 hook 实现的。简单来说就是 HashMap 配合一个双向链表，来存放k-v的顺序。

其核心在于多了两个成员变量
```
static class LinkedHashMapEntry<K,V> extends HashMap.Node<K,V> {
    LinkedHashMapEntry<K,V> before, after;
    LinkedHashMapEntry(int hash, K key, V value, Node<K,V> next) {
        super(hash, key, value, next);
    }
}

/**
* 头部是最老的数据 
*/
transient LinkedHashMapEntry<K,V> head;

/**
* 尾部是最新的数据
*/
transient LinkedHashMapEntry<K,V> tail;

```

### 核心代码

在关键位置进行hook：
```
Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
    LinkedHashMapEntry<K,V> p =
        new LinkedHashMapEntry<K,V>(hash, key, value, e);
    linkNodeLast(p);
    return p;
}
```

上篇中 我们分析了，在putVal的时候，会创建一个新的Node，用的就是 HashMap 的 newNode 方法。而这里重写了此方法，因此 putVal 会调用到内部。

```
private void linkNodeLast(LinkedHashMapEntry<K,V> p) {
    LinkedHashMapEntry<K,V> last = tail;
    tail = p;
    if (last == null)
        head = p;
    else {
        p.before = last;
        last.after = p;
    }
}
```

插入时，直接会进行尾插的方式，把新数据插到尾部。

同时hook了get方法，期间调用了方法：
```
void afterNodeAccess(Node<K,V> e) { // move node to last
    LinkedHashMapEntry<K,V> last;
    if (accessOrder && (last = tail) != e) {
        LinkedHashMapEntry<K,V> p =
            (LinkedHashMapEntry<K,V>)e, b = p.before, a = p.after;
        p.after = null;
        if (b == null)
            head = a;
        else
            b.after = a;
        if (a != null)
            a.before = b;
        else
            last = b;
        if (last == null)
            head = p;
        else {
            p.before = last;
            last.after = p;
        }
        tail = p;
        ++modCount;
    }
}
```
将相应的数据移动到尾部。当然了，get 方法是由 accessOrder 来进行控制的。为 true 才会将get也移动元素。

其余就遍历的时候会用自己实现的方法来覆盖HashMap的方法：
```
public void forEach(BiConsumer<? super K, ? super V> action) {
    if (action == null)
        throw new NullPointerException();
    int mc = modCount;
    // Android-changed: Detect changes to modCount early.
    for (LinkedHashMapEntry<K,V> e = head; modCount == mc && e != null; e = e.after)
        action.accept(e.key, e.value);
    if (modCount != mc)
        throw new ConcurrentModificationException();
}
```
整个遍历是在遍历双向链表而非遍历底层的数组。


## LruCache

其核心就是一个 LinkedHashMap，不过完全没有用 eldest 而已。

直接看方法：

### get
```
public final V get(K key) {
    // 不支持null 的key
    if (key == null) {
        throw new NullPointerException("key == null");   
    }

    V mapValue;
    synchronized (this) {
        //尝试获取相应的value，有的话就直接返回好了。
        mapValue = map.get(key);
        if (mapValue != null) {
            hitCount++;
            return mapValue;
        }
        missCount++;
    }

    /*
        * Attempt to create a value. This may take a long time, and the map
        * may be different when create() returns. If a conflicting value was
        * added to the map while create() was working, we leave that value in
        * the map and release the created value.
        */

    //如果没有，尝试让用户自己判断，要不要自己生成一个。
    V createdValue = create(key);
    if (createdValue == null) {
        return null;
    }

    //如果确实自己生成了一个，就把它放进链表中
    synchronized (this) {
        createCount++;
        mapValue = map.put(key, createdValue);
        // 因为 create 时，可能别的地方已经放入了一个数据，因此需要二次判断是否放置位置有数据，
        //如果有数据，那肯定是被别的线程捷足先登，因此要重新还原人家设置的数据
        if (mapValue != null) {
            // There was a conflict so undo that last put
            map.put(key, mapValue);
        } else {
            //如果没有，那么直接扩容
            size += safeSizeOf(key, createdValue);
        }
    }

    if (mapValue != null) {
        //如果用户自己create了一个对象，但是已经被别人捷足先登，那么给出一个回调，告诉用户，你那里的create 不好使了。
        entryRemoved(false, key, createdValue, mapValue);
        return mapValue;
    } else {
        //缩容到最大限制。
        trimToSize(maxSize);
        return createdValue;
    }
}
```

由于 create 方法可能会耗时好长。同时有别的线程来操作因此，要增加一个二次校验。

```
protected void entryRemoved(boolean evicted, K key, V oldValue, V newValue) {}
```

同时 entryRemoved 方法是一个空方法，如果没有实现就不会有任何显示。


