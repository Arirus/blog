# HashMap

HashMap 主要用来存放键值对，它基于哈希表的Map接口实现，是常用的Java集合之一。

JDK1.8 之前 HashMap 由 数组+链表 组成的，数组是 HashMap 的主体，链表则是主要为了解决哈希冲突而存在的（“拉链法”解决冲突）.JDK1.8 以后在解决哈希冲突时有了较大的变化，当链表长度大于阈值（默认为 8）且size<64时，将链表转化为红黑树，以减少搜索时间。

## 底层数据结构分析

HashMap 底层是 数组和链表 结合在一起使用也就是 链表散列。

HashMap 有两个影响其性能的参数：初始容量（initial capacity）和负载因子（load factor）。容量就是哈希表中 buckets 的数量，负载因子就是 buckets 填满程度的最大比例。当 buckets 填充的数目大于 capacity * load factor 时，就需要调整 buckets 的数目为当前的 2 倍。 如果对迭代性能要求很高的话不要把 initial capacity 设置过大，也不要把 load factor 设置太小。

### 默认常量

```
// 默认的 initial capacity —— 必须是 2 的幂
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

// 最大的 capacity
static final int MAXIMUM_CAPACITY = 1 << 30;

// 默认的加载因子： size 超过 capacity*DEFAULT_LOAD_FACTOR 就会resize
static final float DEFAULT_LOAD_FACTOR = 0.75f;

// 使用 tree 而不是 list 的阈值 进化
static final int TREEIFY_THRESHOLD = 8;

// 使用 list 而不是 tree 的阈值 退化
static final int UNTREEIFY_THRESHOLD = 6;

// 当capacity大于64时，才会树化，不然还是链表
static final int MIN_TREEIFY_CAPACITY = 64;
```

### Node结点
```
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;
}
```

### 相关实现

#### 构造函数
```
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}

public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                            initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                            loadFactor);
    this.loadFactor = loadFactor;
    this.threshold = tableSizeFor(initialCapacity);
}

public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}

public HashMap(Map<? extends K, ? extends V> m) {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    putMapEntries(m, false);
}
```

#### hash
hashMap 中自己实现了一个 hash 方法，并没有用每个作为key的对象的提供的haseCode()方法，主要的原因是：

**当capcity过小的时候，对象hashCode的高位无法参与操作，因此可能产生更多的hash碰撞**

因此将高16位与低16位进行异或操作，使全部数位参与。不用 **&** 或者 **|** 的原因在于：虽然所有数位都参与了，但可能有部分位没有用，因为 1 & x == 1，0 | x == 0。


#### tableSizeFor
构造函数中，直接把`tableSizeFor(initialCapacity)`的值作为阈值。而不是我们认为的 CAP * FACTOR，那是因为初始使用resize时，会重新赋值。
```
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

tableSizeFor 保证给出的返回值是比给定 capacity 大的2的n次幂次整数。

#### resize

resize 方法用于初始化或扩容 table 数组的大小。 当 put 时，如果 table 为空，或者 put 完发现当前的 size 已经超过了 threshold ，就会去调用 resize 进行扩容。

resize 大致步骤：

- 初始化的情况：
    - 当未设置 initialCapacity，newCap 赋值为 DEFAULT_INITIAL_CAPACITY，newThr 赋值为 (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY)，按默认的情况也就是 newCap = 16，newThr = 12。。

    - 当设置了 initialCapacity，那么 threshold 也是存在的，就是 tableSizeFor 返回值，是2的n次幂。
    newCap 就会设置为这个 threshold，newThr 就会设置为 newCap * loadFactor，所以一开始的设置threshold 只是偷梁换柱，没什么卵用。

    分配新buckets，即 Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap]; 然后 return newTab; 结束

    这里给出一个初始化的省略方法：
    ```
    final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap==0 && oldThr > 0) // 调用者自己设置了 init cap
            newCap = oldThr;
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        else {               // 调用者自己没设置了 init cap，使用默认的
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }

        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        return newTab;
    }

    ```

- 扩容的情况： 扩容情况没啥好说的，如果超过最大值，则不再扩充，让 threshold = Integer.MAX_VALUE 然后 return oldTab 结束。 没超过最大值，扩容为原来的 2 倍，然后把每个 Node 移动到新的 Node 数组中去。

    给出一个省略的方法：
    ```
        final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&     //容量翻倍
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // 阈值翻倍
        }

        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;

                    //如果当前index只有这一个元素，就把他直接移动到新的 tab 里面；
                    if (e.next == null)       
                        newTab[e.hash & (newCap - 1)] = e;
                    //如果当前的元素是树结点，对其用树的操作方式    
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    //如果不是树结点，同时还有很多bin，那么进行平移。
                    //平移比较讲究，不是把每个bin进行rehash，而是判断hash与原数组长度的关系
                    //如果小于就放到 新数组的 index 处，和原数组对应，同时遍历链表。（因为通常确定index时，如果hash值大于长度，高位是会被抛弃的，现在如果发现如果长度新增加的一位依然没有值，那么他就还在原来的位置）
                    //如果大于就放到 新数组的 index+oldCap 处，同时遍历链表。（长度新增加了一位，那他应该在一个新的位置）
                    //这样我们其实只需要一位的&操作，而不需要全体的重 & 操作，就可以分开两类数据了。
                    //这样整个时间复杂度就是 O(n) ,无须重新计算每个bin在数组的index
                    else { // preserve order
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }
    ```

#### get

其核心方法在于 getNode 
```
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    //判断数组不空，长度大于0，且hash位置有值
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        //当第一个元素， hash 相等 值相等 就认为是
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        //当不止一个元素时，遍历是否相等。
        if ((e = first.next) != null) {
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```

#### put

其核心在于
```
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    //为空就先申请空间
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    //当前结点没有元素，直接插入
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        //第一个结点就是 key 相等（包括key的hash相等），直接返回第一个结点。
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        //如果是树结点，都走树结点的操作。    
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        //如果是链表结点
        else {
            for (int binCount = 0; ; ++binCount) {
                //整个链表都没有相同的key，直接插入，并判断是否树化。
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                //找到了相同的key，就返回这个结点
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        //更新结点的值。
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    //如果 size 超过了 threshold ，重建tab
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

## 小结：

### HashMap 的存储过程 

调用 put(K key, V value) 方法进行存储。首先通过 hash(key) 计算出 key 的 hash 值，其中 hash 方法会将 key 的 hashCode 的高 16bit 与低 16bit 进行异或，得到一个 hash 值。然后通过 (n - 1) & hash 得到 bucket 的下标位置。根据 key 和 hash 值寻找是否已存在节点，如果已存在则更新旧值（是否更新旧值根据 onlyIfAbsent 字段决定），不存在的话则调用 newNode 生成新 Node，并存储起来。在 bucket 中寻找的时候通过遍历链表或者红黑树，在 jdk 1.8 中，当 bucket 中碰撞冲突的元素超过某个限制（默认是 8），则使用红黑树替代链表，从而提高查询速度。

### capcity长度是2的n次幂

每次扩容都是左移1位，同时使用使用 & 操作来确定数组index，如果不是2的n次幂，部分数位可能出现0，也就是数组元素可能永远都不会有链表在上面。

### 怎么保证resize的时候，hash分布依然是均匀的

关键在于（e.hash & oldCap）算是均等的分为两份了

### 线程不安全

- 多线程的put可能导致元素的丢失：
    ```
    if ((e = p.next) == null) { // #1
        p.next = newNode(hash, key, value, null); // #2
        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
            treeifyBin(tab, hash);
        break;
    }
    ```
    两个线程都断到了#1 的位置，在分别执行#2时，后执行的会把新执行的数据给冲掉。

- put和get并发时，可能导致get为null

    线程1执行put时，因为元素个数超出threshold而导致resize，线程2此时执行get，有可能导致这个问题。

死循环的问题在 1.8 版本已经解决了。jdk1.8之后有一个tail指向尾指针，不再是头插法，而是尾插法了。

### 使用自定义类型作为key

一定要重写 hashCode 和 equals 方法。



