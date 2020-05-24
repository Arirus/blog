---
date : 2018-08-22T10:50:40+08:00
tags : ["基础", "Java", "多线程"]
title : "Java查漏补缺"

---

# 写在最前面

这个不打算写成一个系列，毕竟Java基础这块儿如果没有大量时间是些不好，因此我打算写成一个笔记的形式，就是写到哪儿算哪儿，有新的发现或者想法直接更新。可能会欠缺一些条理，等全部写完再进行整理好了。恩，开始吧。

<!--more-->

# equals() 与 hashCode()

equals()方法是用来判断其他的对象是否和该对象相等。在 Object 中，equals() 有如下定义：

```java
public boolean equals(Object obj) {  
    return (this == obj);  
}  
```

即判断两个对象在内存中的地址是否是一个。因为对于 Object 对象来说，比较内容没有意义，因此这里是直接比较内存地址无可厚非，但是对于某些类如 String ，我想直到每个字符是否是相等的，因此需要重写 equals 方法。当 equals 重写时，通常也要重写 hashCode 方法，这个方法返回一个给定对象的哈希值。注意**两个对象相当则他们的哈希值一定相等，如果不相等则他们的哈希值不一定不等**，这是由哈希值的属性决定的，他只是一个指纹。

想要弄明白hashCode的作用，必须要先知道Java中的集合。　　
总的来说，Java中的集合（Collection）有两类，一类是List，再有一类是Set。前者集合内的元素是有序的，元素可以重复；后者元素无序，但元素不可重复。这里就引出一个问题：要想保证元素不重复，可两个元素是否重复应该依据什么来判断呢？
这就是Object.equals方法了。但是，如果每增加一个元素就检查一次，那么当元素很多时，后添加到集合中的元素比较的次数就非常多了。也就是说，如果集合中现在已经有1000个元素，那么第1001个元素加入集合时，它就要调用1000次equals方法。这显然会大大降低效率。
于是，Java采用了哈希表的原理。哈希（Hash）实际上是个人名，由于他提出一哈希算法的概念，所以就以他的名字命名了。哈希算法也称为散列算法，是将数据依特定算法直接指定到一个地址上，初学者可以简单理解，hashCode方法实际上返回的就是对象存储的物理地址（实际可能并不是）。  
这样一来，当集合要添加新的元素时，先调用这个元素的hashCode方法，就一下子能定位到它应该放置的物理位置上。如果这个位置上没有元素，它就可以直接存储在这个位置上，不用再进行任何比较了；如果这个位置上已经有元素了，就调用它的equals方法与新元素进行比较，相同的话就不存了，不相同就散列其它的地址。所以这里存在一个冲突解决的问题。这样一来实际调用equals方法的次数就大大降低了，几乎只需要一两次。  

简而言之，在集合查找时，hashcode能大大降低对象比较次数，提高查找效率！所以如果自定义类要存放到哈希表里（作为 key），那么 hashCode 方法是一定要重写的，不然的话可以不写，同时对于 hashCode 返回的唯一性直接决定了哈希表的的性能。

# 对象的拷贝

new 一个对象和拷贝一个对象，其实并不矛盾。二者用在的场合不同，前者主要是用在创建一个新的实例的情况下，后者用于将一个现有对象的属性赋到另一个对象上面，有点类似于C++的复制构造函数。通常使得一个对象支持拷贝则实现一个 Cloneable 接口：

```java
class A implements Cloneable{  
    private int number;  

    @Override  
    public Object clone() {  
        A a = null;  
        try{  
            stu = (A)a.clone();  
        }catch(CloneNotSupportedException e) {  
            e.printStackTrace();  
        }  
        return stu;  
    }  
}
```

实现 Cloneable 接口，只是实现了Java的浅拷贝，就是说只会拷贝7个基本类型，此外String 和 自定义类型还是以复制引用地址，也就是说新的实例里面的自定义类型（包括String）的成员变量，还是指向（这个词用的不太好，但是没想到别的词）原实例成员变量的内存地址，二者的自定义成员变量共用同一个内存地址。因此我们另寻一个复制的方法，使用输入流来进行复制，使用流进行复制，完全不要考虑内存共享问题，关键是要求拷贝对象要实现 Serializable 接口

```java
public class A implements Serializable{
    private static final long serialVersionUID = 2631590509760908280L;
    ..................
    //去除clone()方法
}

public static <T extends Serializable> T Clone(T obj) {
  T objNew = null;
  try {
    ByteArrayOutputStream out = new ByteArrayOutputStream(); //介质流
    ObjectOutputStream objectOutputStream = new ObjectOutputStream(out); //装饰流
    objectOutputStream.writeObject(obj);
    objectOutputStream.close();

    ByteArrayInputStream in = new ByteArrayInputStream(out.toByteArray());
    ObjectInputStream objectInputStream = new ObjectInputStream(in);
    objNew = (T) objectInputStream.readObject();
    objectInputStream.close();
  } catch (Exception e) {
  }
  return objNew;
}
```

这样便真正实现了一个对象的深拷贝，注意如果成员变量是自定义类型的话同样要实现 Serializable 接口。

# I/O流

I/O流这里真的挺大，如果掰开了揉碎了讲估计要好几篇，这里我们就调最基本的概念好了。I/O流的结构集成图就不再给出了，网上非常多。

首先从功能上来说分为:`输入流`，`输出流`。那么对于新人一个不是问题的问题来了，以什么为参照对象？例如河流的上下游，流水对于上游就是输出流，对于下游就是输入流。这个问题我们可以从类名上得到答案，以输入流为例，几个常见类`ByteArrayOutputStream`,`FileOutputStream`,`ObjectOutputStream` 都是 `XXXOutputStream`，其实就说他们是 `XXX` 输出流，也就是说他们输出 `XXX`。同理输入流。

以上小结 Clone() 方法为例，ObjectOutputStream 调用了 writeObject 方法，就是向流中写入了一个 Object 对象，那么这个流中便存在这个对象，以供别的地方调用。对输入流只能进行读操作，对输出流只能进行写操作，程序中需要根据待传输数据的不同特性而使用不同的流。 所以程序中使用不同的流不是根据“上下游”来选择，而是根据“上下游”要进行的操作来选择。

接着就是按照类型来分，分为：`字符流`,`字节流`。顾名思义，二者主要在于流的单位不同，前者按照字符来算，后者按照字节来算。另外处理对象不同：字节流能处理所有类型的数据（如图片、avi等），而字符流只能处理字符类型的数据。
```java

char[] b = new char[50];

StringWriter stringWriter = new StringWriter();
stringWriter.write("dsdsds1\n");
stringWriter.write("dsdsds2\n");
stringWriter.write("dsdsds3\n");
stringWriter.close();

StringReader stringReader = new StringReader(stringWriter.toString());

Log.i(ARIRUS, "Clone: "+stringReader.read(b));
Log.i(ARIRUS, "Clone: "+String.valueOf(b,0,22));
stringReader.close();

Log:
I/ARIRUS: Clone: 24
I/ARIRUS: Clone: dsdsds1
                 dsdsds2
                 dsdsds
```

针对字符流，仿照上面给一个例子：向输出流中写入数据,在输入流中读取输出流中写入的数据。

## 字节流

字节流中的输入流基类为`InputStream`,它是一个抽象类。ByteArrayInputStream、FileInputStream 是两个最常用的介质流，分别从Byte 数组、和本地文件中读取数据，介质流直接与数据源打交道。ObjectInputStream 和所有FilterInputStream 的子类（BufferedInputStream，DataInputStream等）都是装饰流。装饰流用于包装介质流，增强其特性。其实两类很好判别，因为装饰流的构造函数都要求一个 InputStream/OutputStream 作为参数进行构造。

`OutputStream` 是所有的输出字节流的父类，它是一个抽象类。ByteArrayOutputStream、FileOutputStream 是两种基本的介质流，它们分别向Byte 数组、和本地文件中写入数据。ObjectOutputStream 和所有FilterOutputStream 的子类（BufferedOutputStream，DataOutputStream 等）都是装饰流。

## 字符流

字符流中的输入流基类为`Reader`，它是一个抽象类。CharArrayReader、StringReader 是两种基本的介质流，它们分别将Char 数组、String中读取数据。BufferedReader 很明显就是一个装饰器，它和其子类负责装饰其它Reader 对象。InputStreamReader 是一个连接字节流和字符流的桥梁，它将字节流转变为字符流。FileReader 便是继承了InputStreamReader，在其源代码中使用了将FileInputStream 转变为Reader 的方法。

`Writer` 是所有的输出字符流的父类，它是一个抽象类。CharArrayWriter、StringWriter 是两种基本的介质流，它们分别向Char 数组、String 中写入数据。BufferedWriter 是一个装饰器为Writer 提供缓冲功能。OutputStreamWriter 是OutputStream 到Writer 转换的桥梁，它的子类FileWriter 其实就是一个实现此功能的具体类，功能和使用和OutputStream 极其类似。

## 基类字节流源码简析

```java
public abstract class InputStream implements Closeable {
  //返回读取字节的数量 -1 未成功
  public abstract int read() throws IOException;

  //返回读取成功的数量
  public int read(byte b[]) throws IOException {
    return read(b, 0, b.length);
  }
  // 调用了实现的read 方法
  public int read(byte b[], int off, int len) throws IOException {
    ...
    for (; i < len ; i++) {
        c = read();
        if (c == -1) {
            break;
        }
        b[off + i] = (byte)c;
    }
    ...
  }
}

public abstract class OutputStream implements Closeable, Flushable {
  //写入一个字节 -128~127
  public abstract void write(int b) throws IOException;
  public void write(byte b[]) throws IOException {
    write(b, 0, b.length);
  }
  public void write(byte b[], int off, int len) throws IOException {
    ...
    for (int i = 0 ; i < len ; i++) {
        write(b[off + i]);
    }
    ...
  }
}
```

输入流主要是留 read 方法给子类进行实现。输出流主要是留 write（int）方法给子类进行实现，性质都是一样的。

# 内部类相关

Android 中最常用的三个内部类 成员内部类 静态内部类 匿名内部类。
很多人不建议使用“成员内部类”和“静态内部类”，我觉得要看情况，确实如果写在一起可能不方便阅读，但是如果内部类和外部类存在逻辑上的联系，写在一起也未尝不可。例如 RecyclerView ，其 Adapter 和 ViewHolder 都是以静态内部类的形式存在于 RecyclerView 的内部，同时还有一个 Recycler 成员内部类在其中，所以不一定要避免这种写法。还有一点需要注意：非静态内部类会持有外部类的一个引用，这也是为啥非静态内部类可以直接使用外部类的成员变量包括私有的。因此，如果在一个 Activity 中，使用了内部类，注意在 onDestory 时要释放，不然可能会产生内存泄漏。

# synchronized volatile 与 单例模式

## synchronized 与 volatile

synchronized 是 java 同步机制的一个缩影。简单的理解就是使用 synchronized 修饰的方法，则整个方法会被锁保护起来。即同一时间内只有一个线程可以访问到函数内部。保证同一时间只有一个线程拿锁。volatile 不同于 synchronized，并不是应用在上锁中，而是同步机制中免于上锁的一个操作，通知 JVM 被 volatile 修饰的域是可能被另一个线程并发更新的。并发编程中的三个特性：原子性，可见性和有序性。一旦一个共享变量（类的成员变量、类的静态成员变量）被volatile修饰之后，那么就具备了两层语义：

    保证了不同线程对这个变量进行操作时的可见性，即一个线程修改了某个变量的值，这新值对其他线程来说是立即可见的。
    禁止进行指令重排序。
因此 volatile 不能保证原子性（atomic包的类支持），可以保证可见性和有序性。其常用在多线程的标志位上，这样一个线程将标志位修改后，别的线程也会立即可见。

## 单例模式

上面说了这么多，以单例模式为例来做一个简单说明：

```Java
//懒汉式 只有在调用的时候才会生成一个实例
public class Singleton {

  private static Singleton INSTANCE;
  private Singleton() {}

  public static Singleton getInstance() {
    if (INSTANCE == null) INSTANCE = new Singleton();
    return INSTANCE;
  }  
}
```

这种懒汉式是非线程安全的，多线程下可能会出现问题。因此可以使用同步加锁保证，同一时刻只有一个线程可以拿锁。之所以要使用 Double Check 的方式，是由于当一个线程在同步块儿内，别的线程在外等候，当线程出了同步块儿，另一个线程进入如果不进行判空校验会生成另一个实例，因此需要再一次判空。

```Java
public static Singleton getInstance() {
  if (INSTANCE == null)
    synchronized (Singleton.class) {
      if (INSTANCE == null)
        INSTANCE = new Singleton();
    }
  return INSTANCE;
}
```

上述还有一个问题 `INSTANCE = new Singleton()` 在赋值时进行了三步操作：

    1.给 Singleton 分配空间；
    2.生成新的 Singleton 实例；
    3.将 `INSTANCE` 指向新分配的空间
由于第2步是依赖于第1步的，不会被重排到1之前，但是第3步和第2步没有依赖关系可能被重排到2之前，因此可能的顺序为1-3-2。当第3步完成时，意味着已经赋值完成 INSTANCE 非空，这时候如果有新的线程来到 getInstance 方法，那么会直接返回 INSTANCE 对象，因此是极有可能产生异常，毕竟返回的是一个尚未初始化的内存地址。因此通常，为了避免这种重排序，会给 INSTANCE 加上 volatile 限定。完整的代码如下：

```Java
private volatile static Singleton INSTANCE; //声明成 volatile
private Singleton (){}
public static Singleton getSingleton() {
  if (INSTANCE == null) {
    synchronized (Singleton.class) {
      if (INSTANCE == null)
        INSTANCE = new Singleton();
    }
  }
  return instance;
}
```

除了上面的懒汉式写法（第一次调用时才会初始化），还有饿汉式写法（第一次加载类时便初始化），使用了 static final 特性，第一次加载初始化，保证线程安全。

```Java
private static final Singleton INSTANCE = new Singleton();
private Singleton (){}
public static Singleton getInstance() {
  return INSTANCE;
}
```

更好的是使用静态内部类，既可以做到懒加载，也可以做到线程安全：

```java
private Singleton(){}

private static class Holder{
  private static final Singleton SINGLETON = new Singleton();
}

public static Singleton getInstance() {
  return Holder.SINGLETON;
}
```

首先，由于 Singletone 中没有 static final 变量，因此第一加载类时，不会生成实例，所以是非饿汉式的。再次，当调用 getInstance 方法时，会加载 Singleton.Holder 类，此时才会生成 Holder.INSTANCE 变量，所以是懒汉式线程安全的单例模式。

最后一个，也是我最常用的写法是使用枚举，创建枚举默认就是线程安全的。

```java
public enum Singleton{
    INSTANCE;
}
```

# try finally 与 return 的关系

对于含有 return 语句的情况，这里我们可以简单地总结如下：try 语句在返回前，将其他所有的操作执行完，保留好要返回的值，而后转入执行 finally 中的语句，而后分为以下三种情况：

    情况一：如果 finally 中有 return 语句，则会将 try 中的 return 语句”覆盖“掉，直接执行 finally 中的 return 语句，得到返回值，这样便无法得到try之前保留好的返回值。

    情况二：如果 finally 中没有 return 语句，也没有改变要返回值，则执行完 finally 中的语句后，会接着执行 try 中的 return 语句，返回之前保留的值。

    情况三：如果 finally 中没有 return 语句，但是改变了要返回的值，这里有点类似与引用传递和值传递的区别，分以下两种情况，：

        1）如果 return 的数据是基本数据类型或文本字符串，则在 finally 中对该基本数据的改变不起作用，try 中的 return 语句依然会返回进入 finally 块之前保留的值。

        2）如果 return 的数据是引用数据类型，而在 finally 中对该引用数据类型的属性值的改变起作用，try 中的 return 语句返回的就是在 finally 中改变后的该属性的值。

# 序列化

所谓序列化就是**把对象转换为字节序列的过程**，我们知道所谓的对象是当程序运行时，存在内存中的数据，那么如果我想把这些内存中的数据存储起来，就需要用到序列化。常见的几种需要序列化的情况：

- 写入文件和底层的流，或者保存到数据库中
- 通过 Socket 在网络上传输。

注意，我们通常使用的HTTP协议来传输时，是不需要的，因为HTTP是文本协议，因此无需序列化。那么序列化的逆过程就是被称为反序列化。这里我们直接给出一个 POJO 类，来说明序列化的用处。

```java
//Pojo 定义
public class Pojo {
  String mStr1;
  String mStr2;
}

//序列化与反序列化来读取流中的数据
Pojo pojo = new Pojo("STTTT1","STTTTTT2");

try{
  //序列化
  ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
  ObjectOutputStream outputStream = new ObjectOutputStream(byteArrayOutputStream);
  outputStream.writeObject(pojo);

  //反序列化
  byte[] bytes = byteArrayOutputStream.toByteArray();
  ByteArrayInputStream byteArrayInputStream = new ByteArrayInputStream(bytes);
  ObjectInputStream objectInputStream = new ObjectInputStream(byteArrayInputStream);
  pojo = (Pojo) objectInputStream.readObject();

  Log.i("PojoTest", pojo);
}catch (IOException e){
  e.printStackTrace();
} catch (ClassNotFoundException e) {
  e.printStackTrace();
}

//log：
java.io.NotSerializableException: cn.arirus.versioncomp.ipc.Pojo
```

程序会直接抛出异常告诉我们，Pojo 类不能序列化。解决办法就是使得 Pojo 类实现 Serializable 接口。还有几点需要注意

- 当变量使用 transient 关键字修饰或者为静态变量时，不会被序列化的。
- 需要显式定义一个静态常量来表示序列化的版本，避免跨编译器编译或者因修改 Pojo 类属性造成的数据反序列化对应不上造成崩溃。

例如：ANY-ACCESS-MODIFIER static final long serialVersionUID = 42L;

# 泛型

## 泛型简述
先说一点，为什么要有泛型，使用 Object 不是也可以的么？用 Object 参数来代替泛型有两点不好：

- 无法在编译时进行校验参数类型
- 使用时必须进行强转，不然会得到 Object 类型

简单的说，泛型是**属性类型参数化**，属性的类型不在是一个固定的类（String, Integer），而是可以通过实例化传入的类型。

泛型不管是在 Java 还是 Android 开发中都是很常见的，例如 RecyclerView 的 ViewHolder 在被使用到 Adapter 时就是以泛型的形式来实例化的。常见的泛型使用包括**泛型类**，**泛型方法**和**泛型接口**，相信大家在开发的时候都见到过，这里就不深入讨论了。

## 通配符
Java 中，我们使用 `<T>` 表示泛型类型，还有 `<?>` 来表示通配符。
```java
class Base{}

class Sub extends Base{}

Sub sub = new Sub();
Base base = sub; // ok

List<Sub> lsub = new ArrayList<>();
List<Base> lbase = lsub; // error 这是个 Base 的列表 不能指向 Sub 列表的引用
List<? extends Base> lbase = lsub; // ok 这是个 Base 及其所有子类的列表 可以指向 Sub 列表的引用
```
之所以会出现上面的情况，其实是：虽然 Sub 是 Base 的子类，但 List<Sub> 和 List<Base> 没有继承关系。

所以，**通配符的出现是为了指定泛型中的类型范围**。常用的通配符的表示有如下三种：

- <?> 被称作无限定的通配符。
- <? extends T> 被称作有上限的通配符。
- <? super T> 被称作有下限的通配符。

### 无限定通配符
无限定通配符常常用在：**不关心容器类对象类型，只关心容器状态**，读取，可删的情况下。
```java
List<?> list = new ArrayList<>();
list.add(new Object()); // error 
list.add("ds"); // error
Object o = list.get(0); // ok
```
这里使用无限定通配符言下之意就是：这个就是一个 List，至于里面参数类型你不用知道，你就对 List 操作就好了。注意使用无限定通配符不同于 `List list = new ArrayList<>()` ，后者是一个 Object 的 List，可以存入任何数据，不属于通配符的操作范畴。List<?> 表示未知类型的列表，而 List<Object> 表示任意类型的列表。

### 上限通配符
上限通配符用在：**容器类对象为某个类及其子类**，可读，可删父类实例，不能写入的情况下。
```java
List<? extends Base> list = new ArrayList<>(2);
list.add(base); // error
list.add(sub); // error
Base base = list.get(list.size()-1);//ok 但是会越界
```
类似无限定通配符：这是一个元素类为 Base 或子类的 List，你不能放入任何 Base 以及子类的对象，只能读取，读取类型为 Base。

### 下限通配符
上限通配符用在：**容器类对象为某个类及父类**，不可读，不可删，能写入子类实例的情况下。
```java
List<? super Base> list = new ArrayList<>(2);
list.add(sub); //ok
list.add(base); //ok
Base sub1 = list.get(0); //error
```
因此简单总结就是：**如果要求参数是返回一个具体类型的容器类时，应该使用上限通配符修饰；如果参数是一个可以放置“足够多”数据的容器，应该使用下限通配符，上限通配符决定可读取的边界，下限通配符决定可写入的边界**。
我们以 Rxjava 来举例：
```java
io.reactivex.Observable.just(new Sub()).subscribe(new Consumer<Base>() {
    ...
});

public final Disposable subscribe(Consumer<? super T> onNext) ;
```
这里上游传入的是一个子类，但是下游却是按照父类的形式来进行消费。T 就相当于 Base，这个匿名 Consumer 可以处理所有 Base 父类的数据自然能处理 Sub。

```java
public boolean addAll(int index, Collection<? extends E> c) ;
```
这个是 ArrayList.addAll 的方法签名，这个里面 Collection 就用了上限通配符，因为要处理 Collection 中元素的数据，所以一定要上限通配符，读出每一个 E，再进行操作。

## 泛型擦除
Java 泛型信息只存在于代码编译阶段，在进入 JVM 之前，与泛型相关的信息会被擦除掉，专业术语叫做类型擦除。
就是说对于下面的代码：
```java
List<String> l1 = new ArrayList<String>();
List<Integer> l2 = new ArrayList<Integer>();

System.out.println(l1.getClass() == l2.getClass());
```
会显示 true，这个就是类型擦除。

# 
# Type
