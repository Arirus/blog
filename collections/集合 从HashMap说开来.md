# 从HashMap说开来

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



