---
date : 2018-04-24T13:21:10+08:00
tags : ["Dagger2", "Android", "用法"]
title : "Dagger2 用法以及一些思考 基础篇"

---

本次使用说明是以 GitHub 上 Dagger 仓库 [2.15](https://github.com/google/dagger)版本来进行的分析，同时以[官方网站](https://google.github.io/dagger/users-guide)文档作为参考。如后续版本有较大改变，恕不另行修改

# 为什么要写这个系列

Dagger 可以说是我用过的最不好理解的第三方开源库之一。其不好理解在于两个方面：其一，他不是针对业务逻辑提出的更方便解决方案。平时我们使用某个第三方库，大部分是针对业务逻辑的，Rxjava可以方便切换线程，OkHttp方便网络请求，如果我们不用这两个也同样可以使用Handler，UrlConnection 同样可来进行处理不过会较为麻烦，但如果不使用 Dagger 几乎不会造成任何的业务困难。其二，使用 apt 意味着需要 build 来生成响应的中间代码，对于理解上又多了一层困难，毕竟不像 ButterKnife 写就完事儿了。既然如此为何还要使用？两个字：“解耦”。 Dagger 是对于控制反转（Inverse of Control）思想的实现，对于系统的解耦有更高的提升（后面会有此处的详细分析）。分析下 Dagger 对于实现构造一个Pure Clean Architecture 是有帮助的。
<!--more-->

# 控制反转与依赖注入

## 控制反转(Inverse of Control)

控制反转是面向对象编程中的一种设计原则，可以用来减低代码之间的耦合度。首先对于耦合度要有一个正确的认识，不能谈“耦”色变，耦合将一个个模块连接起来，使其更像是一个有机的整体，而不是一盘散沙。但是过高的耦合会使得系统变得臃肿，难以维护，牵一发而动全身，所以将耦合降低到一个“合适的程度”是十分必要的。控制反转就可以有效的降低系统的耦合。

![控制反转对比如图](/img/IoC.png)

可以看到在未加入IoC部分时，主模块运行到某一时刻会主动创建模块n，无论是创建还是使用模块n，控制权都在自己手上。但是添加了IoC模块后当主模块运行到需要模块n的时候，IOC容器会主动创建一个模块n注入到主模块需要的地方。主模块获得依赖模块n的过程,由主动行为变为了被动行为，控制权颠倒过来了，这就是“控制反转”这个名称的由来。
通过IoC容器就将各个模块之间的耦合转移到了模块与IoC之间的耦合，降低了修改了某一模块同时影响到另一个模块的几率。

## 依赖注入(Dependency Injection)

本质上依赖注入是实现控制反转的一种方式，所谓依赖不过是持有变量的意思。

```java
public class Gentleman{
    public Wife mWife;

    public Gentleman(){
        mWife = new Wife();
    }
}
```

这样我们便说，Wife 是 Gentleman 的一个依赖。如果这样定义，那么一个 Gentleman 和一个 Wife 在构造时便深度绑定相当于硬编码，但如果我们提供 wife 的 Setter 方法，便不会出现这样的情况。同时也被称为依赖注入。

# Dagger2 基本用法

在了解上述概念以后，我们可以开始 Dagger 的用法了。

## 声明依赖

Dagger2 拥有两种注入方式 声明依赖 和 配置依赖(抱歉没有找到更好的译名，这个译名是我自己想的。。)。我们先看声明依赖，是最简单的一种注入方式。由于Dagger主要是使用注解来进行依赖关系的声明，因此我们直接看各个注解内容。

### @Inject

Inject是Java依赖注入标准提出的注解 [Java 依赖注入标准（JSR-330）简介](https://blog.csdn.net/dl88250/article/details/4838803) 有对其相应的解释这里不再多言。
所谓声明依赖，就是直接通过 @Inject 注解，标注出依赖对象。当标注到构造函数时，Dagger 便使用其来产生依赖对象。

```java
public class A {

  @Inject
  public A() {  }

  public void print(){
    Log.i(ARIRUS, "print: A ");
  }
}

//rebuild 后生成的Factory代码，以提供A的实例
public final class A_Factory implements Factory<A> {
  private static final A_Factory INSTANCE = new A_Factory();

  @Override
  public A get() {
    return new A();
  }

  public static A_Factory create() {
    return INSTANCE;
  }
}
```

### @Component

Component是用于接口和抽象类的注解，从一组模块中为其生成一个完整的，依赖注入的实现。每个Component至少含有一个抽象方法，Component的方法签名必须含有Provider 或者 MembersInjector。

#### Provision methods

对于Provision methods 方法，必须返回Inject修饰的类型或者Provider类型，例如

    SomeType getSomeType();
    Provider<SomeType>  getSomeTypeProvider();
    Lazy<SomeType> getLazySomeType();

#### Members-injection methods

成员注入方法只有一个参数并且将依赖注入到参数中每个被@Inject修饰的字段或者方法。方法可以返回空或者返回其本身用于链式结构。

    void injectSomeType(SomeType someType);
    SomeType injectAndReturnSomeType(SomeType someType);
这里我们使用第二种方法来定义一个Component

```java
@Component()
public interface Main2Component {
  void inject(Main2Activity activity);
}

```

这样便会将 Main2Activity 所有由@Inject修饰的变量和方法注入依赖

```java
public class Main2Activity extends AppCompatActivity {

  public static final String ARIRUS = "Main2Activity";

  public static void start(Context context) {
      Intent starter = new Intent(context, Main2Activity.class);
      context.startActivity(starter);
  }

  @Inject
  A mA;

  @Override protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main2);

    DaggerMain2Component.create().inject(this);
  }
}
```

编译后的代码如下所示

```java
public final class DaggerMain2Component implements Main2Component {
  private DaggerMain2Component(Builder builder) {}

  public static Builder builder() {
    return new Builder();
  }

  public static Main2Component create() {
    return new Builder().build();
  }

  @Override
  public void inject(Main2Activity activity) {
    injectMain2Activity(activity);
  }

  private Main2Activity injectMain2Activity(Main2Activity instance) {
    Main2Activity_MembersInjector.injectMA(instance, new A());
    return instance;
  }

  public static final class Builder {
    private Builder() {}

    public Main2Component build() {
      return new DaggerMain2Component(this);
    }
  }
}
//Main2Activity 编译后
public final class Main2Activity_MembersInjector implements MembersInjector<Main2Activity> {
  private final Provider<A> mAProvider;

  public Main2Activity_MembersInjector(Provider<A> mAProvider) {
    this.mAProvider = mAProvider;
  }

  public static MembersInjector<Main2Activity> create(Provider<A> mAProvider) {
    return new Main2Activity_MembersInjector(mAProvider);
  }

  @Override
  public void injectMembers(Main2Activity instance) {
    injectMA(instance, mAProvider.get());
  }

  public static void injectMA(Main2Activity instance, A mA) {
    instance.mA = mA;
  }
}
```

由于Main2Activity中有使用@Inject修饰的变量，因此生成了相应的MembersInjector，并且由DaggerMain2Component调用injectXX来进行注入。不过我们发现在

    Main2Activity_MembersInjector.injectMA(instance, new A());
调用中直接new了一个A的实例，并没有A_Factory来获得，因此我们可以猜测由于A的Inject方法单一，无需配置因此不需要使用A_Factory来生成实例。

## 配置依赖

对于上面一种情况，会有几种情况不适用：

- @Inject注解参数类型为接口
- 第三方类的构造函数无法被@Inject注解
- 需要配置的对象

```java
//自定义接口
public interface inter {
  void print();
}
//自定义实现类B
public class B implements inter {

  String mString;

  @Inject
  public B() {
    mString = "";
  }

  public void print(){
    Log.i(ARIRUS, "print: B "+mString);
  }
}
//注入依赖
public class Main2Activity extends AppCompatActivity {

  public static final String ARIRUS = "Main2Activity";

  public static void start(Context context) {
      Intent starter = new Intent(context, Main2Activity.class);
      context.startActivity(starter);
  }

  @Inject
  inter mInter;

  @Override protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main2);

    DaggerMain2Component.create().inject(this);
  }
}
//inter 无法提供 依赖生成失败
Error:(19, 8) 错误: com.arirus.daggerapplication.bean.inter cannot be provided without an @Provides- or @Produces-annotated method.
```

原因也好理解，对于一个接口你不知道他的具体实现类是什么，类的构造函数是什么，因此需要生成的依赖则无法确定。同理第三方的类。最后一种是这样的情况：还是以`B`类型为例，如果其构造函数改为：

    public B(String str)
默认的传入参数是无法确定的，因此同样无法使用@Inject来生成默认依赖对象。因此在我理解`声明依赖`使用范围较窄。

## Module 与 Provides

之所以将这两注解放到一起来说，是因为所有的@Provides 都必须属于 @Module 不能单独存在。

```java
@Module
public class Main2Module {
  @Provides
  public inter provideB(){ return  new B(); }
}

//编译后
public final class Main2Module_ProvideBFactory implements Factory<inter> {
  private final Main2Module module;

  public Main2Module_ProvideBFactory(Main2Module module) {
    this.module = module;
  }

  @Override
  public inter get() {
    return Preconditions.checkNotNull(
        module.provideB(), "Cannot return null from a non-@Nullable @Provides method");
  }

  public static Main2Module_ProvideBFactory create(Main2Module module) {
    return new Main2Module_ProvideBFactory(module);
  }

  public static inter proxyProvideB(Main2Module instance) {
    return Preconditions.checkNotNull(
        instance.provideB(), "Cannot return null from a non-@Nullable @Provides method");
  }
}
```

这就相当于我们提供了inter的生成方法，交由module。同样@Provides 方法拥有一个自己的依赖也是可以的（It’s possible for @Provides methods to have dependencies of their own.）

```java
@Module
public class Main2Module {
  //由于B的构造方法有Inject注解修饰，因此可有提供的构造函数来生成依赖实例
  @Provides
  public inter provideB(B b){ return  b; }
}
```

当有相关的提供模块和方法，还要和相应的Component来进行关联。不然作为链接MemberInjector与Provider的桥梁，Component怎么知道从那个Module获取到相应的依赖实例？

```java
@Component(modules = {Main2Module.class})
public interface Main2Component {
  void inject(Main2Activity activity);
}
```

这样一个基本的配置依赖就实现了。注意如果配置的module无法自动生成，则需要手动调用setter来设置相应的module。

```java
@Module
public class Main2Module {

  public Main2Module(String s) {
  }

  @Provides
  public inter provideInter(B b){ return  b; }

  @Provides
  public B provideB(){
    return new B();
  }
}
//Main2Activity 中的调用，不同于上面，无法生成默认Main2Module对象，因此无create方法。
DaggerMain2Component
    .builder()
    .main2Module(new Main2Module("ss"))
    .build().inject(this);
```

# 小结

Dagger2 最基本的用法就是上面所示，Component 是桥梁，调用生成的MembersInjector 开进行注入，而依赖实例的来源是由 @Inject 修饰的构造函数，或是Module中Provides提供的依赖实例。
![Dagger基本原理](/img/dagger基本原理)