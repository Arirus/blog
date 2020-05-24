---
date : 2018-11-14T17:30:52+08:00
tags : ["底层", "Android", "跨进程通信"]
title : "Android 跨进程通信 Parcel "

---

# 写在最前面

今天开始，我们就开始写新的系列：Android 的跨进程通信。说实在，我写这里特别的慌，从现在开始就要开始涉及 Android 的底层的一些东西了。作为一个优秀的移动应用开发，这些东西是进阶的必须要掌握的，那我们迎着头皮也要啃下去，不能总呆在舒适区。因此这里东西比较复杂，因此我们要一步一个脚印前进，本篇就从跨进程最基础的"数据载体"--Parcel 开始。好了我们开始。

<!--more-->

# 为什么需要 Parcel

我们知道，数据类型分为 `基本类型` 和 `自定义类型`，在调用函数中传递参数时基本类型直接传递真实值就可以了，自定义类型处于性能的考虑不是传递的实例，而是原先参数的引用，其实就是一个内存地址（类似于C++的指针）。如果调用都是在同一个进程中这样做当然没有问题，但是对于跨进程程序，B 进程不知道 A 进程的情况，你给它 A 进程的内存地址它也访问不了。因此直接对于跨进程程序是无法直接传递自定义类型的参数的。

Parcel 就是针对这种情况而生的，他将需要传递的数据进行打包，传递到目标进程，再将数据还原出来，这样便完成了进程间的数据传递。

# Parcel 基本说明

我们先来看看，文档上是怎么对 Parcel 进行说明的。

> Container for a message (data and object references) that can be sent through an IBinder.Parcel is not a general-purpose serialization mechanism. it is not appropriate to place any Parcel data in to persistent storage

文档上对其定义十分清晰了，就是一个携带信息的容器用在 IBinder 中，方便跨进程间传递数据。同时，也说明了最重要的一点就是：他是用来在IPC传输的高性能API，不是普通意义上的序列化，因此也完全适用于进行数据的持久化，就像没有 serialVersionUID 来做控制。这里并没有说，和 serialized 相比的效率的问题，所以我们暂且按下不表。

## 处理类型

还是围绕 Parcel 处理类型来说，主要分为下面几种类型：

- Primitives 基本类型 int double String 等
- Primitive Arrays 基本类型数组
- Parcelables Parcelables类型
- Bundles Bundles 其实也是实现了 Parcelable 的接口
- Active Objects
- Untyped Containers 任意类型的 Java 容器

其中 Active Objects 这个类别我没太看的明白，大致就是用于传递 Binder 和 FileDescriptor 的，文档中也说用的比较少，而且我们这一篇主要是说 Parcel 的机制，那么这里我们就先不细究了。

## 通用方法

现在我们大致知道，Parcel 是个什么玩意儿，就像流一样，可以写入也可以读出，虽然可以读取的数据类型，但那应该有一些通用的方法。确实，下面就是其重要的通用方法:

```java
public static Parcel obtain() //获取一个实例
public final void recycle() //回收实例，之后不可用
public final int dataSize() //获取实例中数据的中长度
public final int dataAvail() //获取可读的数据长度
public final int dataPosition() //获取当前的数据位置值，类似游标
public final void setDataSize(int size) // 设置实例大小
public final void setDataPosition(int pos) //设置读取写入数据的位置
```

看了上面的通用方法，Parcel 的数据模型差不多也比较清楚了，有点类似于 StringBuilder 可以 append 数据，也可从某个位置开始读取和写入数据。好了，我们可以试着使用 Parcel 来传递数据了。

# Parcelable

上面提到了，可以使用 Parcel 传输的类型有一类叫做 Parcelable，我们来看看 Parcelable 是个什么。源码也不多，这里我们就不贴出了，大致上就是如果我们要实现一个 Parcelable 需要实现两个方法：

```java
public void writeToParcel(Parcel dest, int flags);
public int describeContents();
```

其中 describeContents 是返回一个 flag，如果这个 Pojo 类中包含 FILE_DESCRIPTOR ，则返回 CONTENTS_FILE_DESCRIPTOR 否则返回0就行，因此我们都返回0就好了。在 writeToParcel 中的 flag 如果被写入的数据是其他函数的运行结果， 会阻止资源的释放，如果不是的话传入0就行。

我们看，其实也就是需要实现一个 Pojo 类属性向 Parcel 写入的方法就可以，这个方法其实就是关键，我们下文会说到 Parcel 的读取和写入的顺序问题，他们应该是一一对应的。因此除此之外应该有一个 Parcel 为参数的 Pojo 的构造方法，这样才能做到一一对应。一个完成的 Pojo 的 Parcel 实现应该这么写：

```java
public class Pojo implements Parcelable {
  String mStr1;
  String mStr2;

  public Pojo(String str1, String str2) {
    mStr1 = str1;
    mStr2 = str2;
  }

  protected Pojo(Parcel in) {
    mStr1 = in.readString();
    mStr2 = in.readString();
  }

  public static final Creator<Pojo> CREATOR = new Creator<Pojo>() {
    @Override public Pojo createFromParcel(Parcel in) {
      return new Pojo(in);
    }

    @Override public Pojo[] newArray(int size) {
      return new Pojo[size];
    }
  };

  @Override public int describeContents() {
    return 0;
  }

  @Override public void writeToParcel(Parcel dest, int flags) {
    dest.writeString(mStr1);
    dest.writeString(mStr2);
  }

   public String toString() {
    return "Str1:"+mStr1+" Str2:"+mStr2;
   }
}
```

这里，我们除了上面说的三个函数还生成了一个 CREATOR ，那么这个静态变量是干嘛的，我们之后再说。

# Parcel 简单示例

好了，上面也分析了 Parcelabel 的相关内容，这里我们就看看其是怎样的一个传递过程。

## 基本类型传输

```java
Parcel parcel = Parcel.obtain();
parcel.writeInt(i);
parcel.writeString(string);
parcel.writeDouble(d);

parcel.setDataPosition(0);

Log.i("IpcActivity", "Arirus onCreate: "+parcel.readInt());
Log.i("IpcActivity", "Arirus onCreate: "+parcel.readString());
Log.i("IpcActivity", "Arirus onCreate: "+parcel.readDouble());

log:
IpcActivity: Arirus onCreate: 1
IpcActivity: Arirus onCreate: Hello Parcel
IpcActivity: Arirus onCreate: 0.5
```

这里我们就不模仿跨进程了，直接写入再读取好了。这里有两点需要注意：

- Parcel 在写入完数据后，如果要直接读取需要把`游标`移动到数据最开头的地方，或者移动到一个你想要开始的位置，不能直接读取，会没有任何数据的。
- 读取顺序和写入顺序应该是一直的。类别我们刚才说的 StringBuilder 模型，如果顺序不一致，而字段大小不一定相等那么一定会读取乱码的。

## Parcelable 类型传输

```java
Pojo pojo = new Pojo("STTTT1","STTTTTT2");

Parcel parcel = Parcel.obtain();
parcel.writeParcelable(pojo,0);

parcel.setDataPosition(0);

Pojo p = parcel.readParcelable(Pojo.class.getClassLoader());

Log.i("IpcActivity", "Arirus onCreate: "+p.toString());

log:
IpcActivity: Arirus onCreate: Str1:STTTT1 Str2:STTTTTT2
```

和写入读取普通数据一样，调用相应的方法就行了。我们这里来看看这两个读取与写入方法到底干了什么。

### writeParcelable readParcelable

```java
public final void writeParcelable(Parcelable p, int parcelableFlags) {
    if (p == null) {
        writeString(null);
        return;
    }
    writeParcelableCreator(p);
    p.writeToParcel(this, parcelableFlags); //直接调用 Pojo.writeToParcel 来写入到 Parcel 中
}
```

writeToParcel 没啥好说的，直接调用 Pojo 类的方法就解决了。

```java
public final <T extends Parcelable> T readParcelable(ClassLoader loader) {
    Parcelable.Creator<?> creator = readParcelableCreator(loader);
    if (creator == null) {
        return null;
    }
    if (creator instanceof Parcelable.ClassLoaderCreator<?>) {
        Parcelable.ClassLoaderCreator<?> classLoaderCreator =
            (Parcelable.ClassLoaderCreator<?>) creator;
        return (T) classLoaderCreator.createFromParcel(this, loader);
    }
    return (T) creator.createFromParcel(this);
}
```

readParcelable 方法主要逻辑就是先获得一个 Creator，通过调用 Creator.createFromParcel 方法，根据 Parcel 来生成对应的 Parcelable 。因此可以理解这个 Creator 为一个代理，通过它来进行生成。

# 小结

本篇中，我们主要介绍了 Parcel 和 Parcelable ，简单的来说就是 Parcel 是一个运载体，里面拥有需要传输数据的信息。Parcelable 是一种可以用于 Parcel 传输的类型，其本质就是手动调用了各个属性的 Parcel 的读取和写入的方法。本篇的内容到此结束，下篇再见。