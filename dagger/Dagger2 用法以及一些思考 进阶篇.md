---
date : 2018-04-28T15:05:45+08:00
tags : ["Dagger2", "Android", "用法"]
title : "Dagger2 用法以及一些思考 进阶篇"

---

本次使用说明是以 GitHub 上 Dagger 仓库 [2.15](https://github.com/google/dagger)版本来进行的分析，同时以[官方网站](https://google.github.io/dagger/users-guide)文档作为参考。如后续版本有较大改变，恕不另行修改

上一篇中，我们了解控制反转设计原则，注入依赖概念，并学习了Dagger2 的最基本用法，本篇我们来说说 Dagger2 的另外一些重要功能。
<!-- more -->

# @Named 与 @Qualifier

接上一篇的配置依赖，假定现有 @Inject 修饰的是一个接口，那么注入的依赖一定是此接口的一个实例，现在假定继承关系如下：

```java
//自定义接口
public interface inter {
  void print();
}
//自定义实现类A
public class A implements inter {
  @Inject
  publicA() {
  }

  public void print(){
    Log.i(ARIRUS, "print: A ");
  }
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
```

那么在Module 和 Main2Activity的注入中，有如下配置：

```java
@Module
public class Main2Module {
  @Provides
  public static inter provideInterB(){ return  new B(); }

  @Provides
  public static inter provideInterA(){ return  new A(); }
}


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
```

运行编译会报错：

    Error:(24, 8) 错误: com.arirus.daggerapplication.bean.inter is bound multiple times:
    @Provides com.arirus.daggerapplication.bean.inter com.arirus.daggerapplication.modules.Main2Module.provideInterB()
    @Provides com.arirus.daggerapplication.bean.inter com.arirus.daggerapplication.modules.Main2Module.provideInterA()
就是说注入生成依赖实例时有两个返回类型一摸一样的@Provides 方法，不知道该生成哪个。此时 @Named 与 @Qualifier 便可以起作用了。

## @Named

@Named 与 @Qualifier 都属于[Java 依赖注入标准（JSR-330）简介](https://blog.csdn.net/dl88250/article/details/4838803)定义的注解。用于标注依赖实例。

```java
@Module
public class Main2Module {
  @Provides
  @Named("B")
  public static inter provideInterB(){ return  new B(); }

  @Provides
  @Named("A")
  public static inter provideInterA(){ return  new A(); }
}

public class Main2Activity extends AppCompatActivity {
  ...  
  @Inject
  @Named("A")
  inter mInter;
  ...
}
```

这样便限定了注入生成的 `inter` 一定是 `A` 的实例。从而解决了上述问题。
注意：刚才我们所说的都是针对 Component 中 Members-injection methods，就是通过注入到参数中每个被@Inject修饰的字段或者方法并不适用于 Component 的 Provides methods。

## @Qualifier

@Named 注解以字符串的形式来定义每个名字，使用 @Qualifier 元注解可以更好的定义注解以区分依赖。

```java
@Qualifier
@Retention(RetentionPolicy.RUNTIME)
public @interface Color {
  ColorE color() default ColorE.TAN;
  public enum ColorE { RED, BLACK, TAN }
}

@Module
public class Main2Module {
  ...

  //@Named("B")
  @Color(color = Color.ColorE.BLACK)
  @Provides
  public static inter provideInterB(){ return  new B(); }

  @Color(color = Color.ColorE.RED)
  //@Named("A")
  @Provides
  public static inter provideInterA(){ return  new A(); }

  ...
}

public class Main2Activity extends AppCompatActivity {
  ...  
  @Inject
  @Color(color = Color.ColorE.RED)
  inter mInter;
  ...
}
```

Qualifier 与 Named 两个注解的功能是一样的，都是为了区分依赖，更推荐前者。

# @Scope 与 @Singleton

Scope 也是一个元注解，用来确定注入的实例的生命周期的，如果没有使用 Scope 注解，Component 每次调用 Module 中的 provide 方法或 Inject 构造函数生成的工厂时都会创建一个新的实例，而使用 Scope 后可以复用之前的依赖实例。Singleton 是 Scope 的一个实现。
这里我们举个例子来说明：

```java
//定义 MyScope 注解
@Documented
@Retention(RUNTIME)
@Scope
public @interface MyScope {
}

//MyScope 修饰 Component
@Component(modules = {Main2Module.class})
@MyScope
public interface Main2Component {
  void inject(Main2Activity activity);
  A  getA();
  B  getB();
}

//Provides methods 使用 MyScrope 修饰
@Module
public class Main2Module {

  @MyScrope
  @Provides
  public A provideA(){
    return new A();
  }

  @Provides
  public B provideB(){
    return new B();
  }

}

//Activity 中的调用
@Override protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main2);

    Main2Component component = DaggerMain2Component.create();
    for (int i = 0; i < 5; i++) {
      Log.i(ARIRUS, "onCreate: "+component.getA());
      Log.i(ARIRUS, "onCreate: "+component.getB());
    }
}
```

这里直接打印5个新的 A 、B 对象。结果如下

```java
05-02 18:05:44.902 15152-15152/? I/Main2Activity: onCreate: com.arirus.daggerapplication.bean.A@28d434e1
05-02 18:05:44.902 15152-15152/? I/Main2Activity: onCreate: com.arirus.daggerapplication.bean.B@117d4206
05-02 18:05:44.902 15152-15152/? I/Main2Activity: onCreate: com.arirus.daggerapplication.bean.A@28d434e1
05-02 18:05:44.902 15152-15152/? I/Main2Activity: onCreate: com.arirus.daggerapplication.bean.B@331720c7
05-02 18:05:44.902 15152-15152/? I/Main2Activity: onCreate: com.arirus.daggerapplication.bean.A@28d434e1
05-02 18:05:44.902 15152-15152/? I/Main2Activity: onCreate: com.arirus.daggerapplication.bean.B@2dbfc6f4
05-02 18:05:44.902 15152-15152/? I/Main2Activity: onCreate: com.arirus.daggerapplication.bean.A@28d434e1
05-02 18:05:44.902 15152-15152/? I/Main2Activity: onCreate: com.arirus.daggerapplication.bean.B@3dfe3e1d
05-02 18:05:44.902 15152-15152/? I/Main2Activity: onCreate: com.arirus.daggerapplication.bean.A@28d434e1
05-02 18:05:44.902 15152-15152/? I/Main2Activity: onCreate: com.arirus.daggerapplication.bean.B@39b7892
```

可以看到，getA() 得到的都是同一个 A 对象，但是对于getB() 得到的对象都不一样。我们可以看一下编译的生成的文件：

```java
public final class DaggerMain2Component implements Main2Component {
  private Main2Module main2Module;

  private Provider<A> provideAProvider;

  ...

  @SuppressWarnings("unchecked")
  private void initialize(final Builder builder) {
    this.provideAProvider =
        DoubleCheck.provider(Main2Module_ProvideAFactory.create(builder.main2Module));
    this.main2Module = builder.main2Module;
  }

  @Override
  public void inject(Main2Activity activity) {
    injectMain2Activity(activity);
  }

  @Override
  public A getA() {
    return provideAProvider.get();
  }

  @Override
  public B getB() {
    return Main2Module_ProvideBFactory.proxyProvideB(main2Module);
  }
  ...
}

//DoubleCheck 的实现
public final class DoubleCheck<T> implements Provider<T>, Lazy<T> {
  private static final Object UNINITIALIZED = new Object();

  private volatile Provider<T> provider;
  private volatile Object instance = UNINITIALIZED;

  private DoubleCheck(Provider<T> provider) {
    assert provider != null;
    this.provider = provider;
  }

  @SuppressWarnings("unchecked") // cast only happens when result comes from the provider
  @Override
  public T get() {
    Object result = instance;
    if (result == UNINITIALIZED) {
      synchronized (this) {
        result = instance;
        if (result == UNINITIALIZED) {
          result = provider.get();
          /* Get the current instance and test to see if the call to provider.get() has resulted
           * in a recursive call.  If it returns the same instance, we'll allow it, but if the
           * instances differ, throw. */
          Object currentInstance = instance;
          if (currentInstance != UNINITIALIZED && currentInstance != result) {
            throw new IllegalStateException("Scoped provider was invoked recursively returning "
                + "different results: " + currentInstance + " & " + result + ". This is likely "
                + "due to a circular dependency.");
          }
          instance = result;
          /* Null out the reference to the provider. We are never going to need it again, so we
           * can make it eligible for GC. */
          provider = null;
        }
      }
    }
    return (T) result;
  }
  ...
}
```

我们可以看到由于@MyScrope修饰了A的get方法，因此在DaggerMain2Component中，getA返回的不是Main2Module提供的provideA方法，而是在创建对象的时候，持有了一个Provider的对象，而其本身也是调用了DoubleCheck来产生的一个provider对象，我们看到当调用Provider方法的get方法，是一个单例的写法，也就是说如果之前有了唯一的A，继续调用Provider的get方法并不会产生一个新的A的实例，而是返回A的单例对象。这也就是说一个生成的DaggerXXXComponent持有了一个DoubleCheck实例，而后者又持有需注入依赖的实例，因此实例的生命周期就和Component 一致了。这就是说，如果我们要使用@Scrope注解来进行修饰，注意潜在的内存泄漏的风险。
而对于@Singleton 是Dagger提供的一个默认实现的@Scrope注解。本质上和我们自己定义的@MyScope 没什么区别。

```java
@Scope
@Documented
@Retention(RUNTIME)
public @interface Singleton {}
```

# @Subcomponent 与 @Component.dependencies

这两个注解（后一个并不是真正意义上的注解，姑且这么叫吧）比较容易让人搞混，因为从功能上二者都是为了解决依赖复用情况。其意义上更接近OOP的is-a has-a的关系，我们先将代码贴出来再分析二者的不同。
假设有个 Daddy 类，其注入依赖 Car 类。在Activity中来进行注入。

```java
public class Car {
  public String id;
}

public class Daddy implements inter {
  @Inject Car mCar;

  @Override public void print() {
    Log.i(ARIRUS, "print: "+mCar.id);
  }
}

@Component(modules = {DaddyModule.class})
public interface DaddyComponent {
  Daddy inject(Daddy daddy);
}

@Module
public class DaddyModule {
  @Provides Car getCar(){
    Car car = new Car();
    car.id= getClass().getName();
    return car;
  }
}

public class Main2Activity extends AppCompatActivity {

  @Override protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main2);

    Daddy daddy = new Daddy();

    DaddyComponent daddyComponent = DaggerDaddyComponent.create();
    daddyComponent.inject(daddy);

    daddy.print();
  }
}
//打印出
I/Main2Activity: print: com.arirus.myapplication.DaddyModule
```

## @Component.dependencies

现在一个 Mam 类，其内部也有一个 Car ，定义如下

```java
public class Mam implements inter{
  @Inject Car mCar;

  @Override public void print() {
    Log.i(ARIRUS, "print: "+mCar.id);
  }
}
```

但是如果Component还是像DaddyComponent一样，就要自己定义Module 来 @Provides Car 实例，如果我们想从 DaddyComponent 直接获取，即Car的实例，只从 DaddyComponent 来获取我们可以定义MamComponent如下：

```java
@Component(dependencies = DaddyComponent.class)
public interface MamComponent {
  Mam inject(Mam mam);
}
```

使用Component.dependencies 来指定依赖的 Component 。同时在 DaddyComponent 要做相应的修改：

```java
@Component(modules = {DaddyModule.class})
public interface DaddyComponent {
  Daddy inject(Daddy daddy);

  Car getCar();
}
```

添加 Car 的 Provider 方法。这样在inject(mam) 时，Car 的实例便由 DaddyComponent 来提供。
最后就是在Activity中的调用：

```java
Daddy daddy = new Daddy();
Mam mam = new Mam();

DaddyComponent daddyComponent = DaggerDaddyComponent.create();
daddyComponent.inject(daddy);

DaggerMamComponent.builder().daddyComponent(daddyComponent).build().inject(mam);
```

所以，调用的时候是将一个 DaddyComponent 实例，注入到 DaggerMamComponent ，以便使用其的 Car 对象。
我们来看下几个关键的编译后的文件

```java
//DaggerDaddyComponent
public final class DaggerDaddyComponent implements DaddyComponent {
  private DaddyModule daddyModule;
  ...
  @Override
  public Daddy inject(Daddy daddy) {
    return injectDaddy(daddy);
  }

  @Override
  public Car getCar() {
    return DaddyModule_GetCarFactory.proxyGetCar(daddyModule);
  }

  private Daddy injectDaddy(Daddy instance) {
    Daddy_MembersInjector.injectMCar(instance, DaddyModule_GetCarFactory.proxyGetCar(daddyModule));
    return instance;
  }

  public static final class Builder {
    private DaddyModule daddyModule;

    public DaddyComponent build() {
      if (daddyModule == null) {
        this.daddyModule = new DaddyModule();
      }
      return new DaggerDaddyComponent(this);
    }
    ...
  }
}

//DaggerMamComponent
public final class DaggerMamComponent implements MamComponent {
  private DaddyComponent daddyComponent;
  ...
  @Override
  public Mam inject(Mam mam) {
    return injectMam(mam);
  }

  private Mam injectMam(Mam instance) {
    Mam_MembersInjector.injectMCar(
        instance,
        Preconditions.checkNotNull(
            daddyComponent.getCar(), "Cannot return null from a non-@Nullable component method"));
    return instance;
  }

  public static final class Builder {
    private DaddyComponent daddyComponent;

    private Builder() {}

    public MamComponent build() {
      if (daddyComponent == null) {
        throw new IllegalStateException(DaddyComponent.class.getCanonicalName() + " must be set");
      }
      return new DaggerMamComponent(this);
    }

    public Builder daddyComponent(DaddyComponent daddyComponent) {
      this.daddyComponent = Preconditions.checkNotNull(daddyComponent);
      return this;
    }
  }
}
```

两个 DaggerXXXComponent 文件核心部分如上所示。对于DaggerDaddyComponent inject时，直接使用 DaddyModule 实例提供的 getCar 方法，来获取 Car 依赖，进行注入。对于 DaggerMamComponent ，要动态传入 DaddyComponent，最后在inject的方法中，调用了 DaddyComponent 实现类（DaggerDaddyComponent）来获取的 Car 依赖。

## @Subcomponent

加入再有一个Child 类，定义如下：

```java
public class Child implements inter {

  @Inject Car mCar;

  @Override public void print() {
    Log.i(ARIRUS, "print: "+mCar.id);
  }
}
```

其内部也有一个 Car ,如果和 Mam 类似，其没有相应的Module的Provides 方法来提供 Car 的依赖，但不以dependencies的形式，便可以以@Subcomponent形式来做：

```java
@Subcomponent
public interface ChildComponent {

  void inject(Child child);

  @Subcomponent.Builder
  interface Builder{
    ChildComponent build();
  }
}

@Component(modules = {DaddyModule.class})
public interface DaddyComponent {
  Daddy inject(Daddy daddy);

  Car getCar();

  ChildComponent.Builder childComponent();
}

//注入
Child child = new Child();

DaddyComponent daddyComponent = DaggerDaddyComponent.create();
daddyComponent.childComponent().build().inject(child);
```

这样Child 便可以使用，Daddy的所有依赖。然后看下编译过后的文件，ChildComponent 没有生成相应的 DaggerChildComponent 文件，所以主要是 DaggerDaddyComponent 文件：

```java
public final class DaggerDaddyComponent implements DaddyComponent {
  private DaddyModule daddyModule;

  private DaggerDaddyComponent(Builder builder) {
    initialize(builder);
  }

  public static Builder builder() {
    return new Builder();
  }

  public static DaddyComponent create() {
    return new Builder().build();
  }

  @SuppressWarnings("unchecked")
  private void initialize(final Builder builder) {
    this.daddyModule = builder.daddyModule;
  }

  @Override
  public Daddy inject(Daddy daddy) {
    return injectDaddy(daddy);
  }

  @Override
  public Car getCar() {
    return DaddyModule_GetCarFactory.proxyGetCar(daddyModule);
  }

  @Override
  public ChildComponent.Builder childComponent() {
    return new ChildComponentBuilder();
  }

  private Daddy injectDaddy(Daddy instance) {
    Daddy_MembersInjector.injectMCar(instance, DaddyModule_GetCarFactory.proxyGetCar(daddyModule));
    return instance;
  }

  public static final class Builder {
    private DaddyModule daddyModule;

    private Builder() {}

    public DaddyComponent build() {
      if (daddyModule == null) {
        this.daddyModule = new DaddyModule();
      }
      return new DaggerDaddyComponent(this);
    }

    public Builder daddyModule(DaddyModule daddyModule) {
      this.daddyModule = Preconditions.checkNotNull(daddyModule);
      return this;
    }
  }

  private final class ChildComponentBuilder implements ChildComponent.Builder {
    @Override
    public ChildComponent build() {
      return new ChildComponentImpl(this);
    }
  }

  private final class ChildComponentImpl implements ChildComponent {
    private ChildComponentImpl(ChildComponentBuilder builder) {}

    @Override
    public void inject(Child child) {
      injectChild(child);
    }

    private Child injectChild(Child instance) {
      Child_MembersInjector.injectMCar(
          instance, DaddyModule_GetCarFactory.proxyGetCar(DaggerDaddyComponent.this.daddyModule));
      return instance;
    }
  }
}
```

可以看到没有生成 DaggerChildComponent，而是生成了 ChildComponentImpl 以内部类的形式存在于 DaggerDaddyComponent 。同时可以看到，不通过与 DaggerMamComponent 的injectMCar 调用的 daddyComponent.getCar()，而是直接调用的 DaddyModule_GetCarFactory.proxyGetCar(DaggerDaddyComponent.this.daddyModule)，因此二者最大的区别就是对于@Component.dependencies，只能使用暴露出的接口来进行注入，而用@SubComponent 则是完全继承，无论是否暴露出接口，只要在Module的中定义了相应的Provider，就可以直接进行注入。

# 总结

本篇其实是比较几个在Dagger中常用注解的异同，算是对于上篇的一个补充，下一篇说说在开发中的具体应用。