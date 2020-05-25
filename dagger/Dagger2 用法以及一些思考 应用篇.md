---
date : 2018-05-25T14:36:25+08:00
tags : ["Dagger2", "Android", "用法"]
title : "Dagger2 用法以及一些思考 应用篇"

---

本次使用说明是以 GitHub 上 Dagger 仓库 [2.15](https://github.com/google/dagger)版本来进行的分析，同时以[官方网站](https://google.github.io/dagger/users-guide)文档作为参考。如后续版本有较大改变，恕不另行修改

# Dagger 在项目中的使用

前两篇中我们了解 Dagger 的基本用法和高级用法，本篇中，我们通过一些开源项目和我司自己的项目为例来说下具体的使用。
<!--more-->
通常我们在开发一个App时，有全局单例的变量，例如 AccountMgr 用来管理账户信息相关，DataMgr 负责管理本地数据和网络请求数据。这些变量往往是不会随着不同 Presenter 请求信息而改变，通常是在 App 初始化的时候，他们就构造完成。还有局部单例变量，最常见的就是每个 View 层配套的 Presenter 变量。使用上篇说的 @Singleton 和 @Scope 可以很好的实现这样的功能。
因此我们可以把全局变量定义到 AppComponent ，并在相应的依赖的 module 来提供单例。

```java

@Singleton
@Component(modules = AppModule.class)
public interface AppComponent {

  ApiClient apiClient();

  AccountMgr accountMgr();

  @Component.Builder
  interface Builder{
    @BindsInstance
    AppComponent.Builder app(AppParent app);
    AppComponent build();
  }
}

@Module
public class AppModule {

  @Provides
  @Singleton ApiClient getApiClient(AccountMgr accountMgr){
    ...
    return new ApiClient();
  }

  @Provides
  @Singleton AccountMgr getAccountMgr(SharedPreferences sharedPreferences){
    return new AccountMgr();
  }
}
```

这里面 @BindsInstance 是用来代替有参数的 AppModule 的构造函数。

# 当前开源项目中使用Dagger的两种方式

上面说的是针对App的全局的单例变量，而对于局部的单例变量，会有两种常见的实现方式。上一篇最后的部分我们分析了 Subcomponent 与 Component.dependencies 的区别。而大多依赖注入也是通过这两种方式来进行的。

## 以 Subcomponent 方式

Subcomponent 我们说过，是不会产生 DaggerXXXComponent，而是定义的 Subcomponent.Builder 由 Component 调用相应的 build() 来获得相应 SubComponent Impl实例。

```java
@ActivityScope
@Subcomponent(
        modules = SplashActivityModule.class
)
public interface SplashActivityComponent {

    void inject(SplashActivity splashActivity);

    @Subcomponent.Builder
    interface Builder{
        SplashActivityComponent build();
    }
}

@Module
public class SplashActivityModule {
    private SplashActivity splashActivity;

    public SplashActivityModule(SplashActivity splashActivity) {
        this.splashActivity = splashActivity;
    }

    @Provides
    @ActivityScope
    SplashActivity provideSplashActivity() {
        return splashActivity;
    }

    @Provides
    @ActivityScope
    SplashActivityPresenter
    provideSplashActivityPresenter() {
        return new SplashActivityPresenter();
    }
}

@Singleton
@Component(modules = AppModule.class)
public interface AppComponent {
   SplashActivityComponent.Builder splashActivityComponent();
   ...
}
```

这样每个 SubComponent 都是“继承” AppComponent。不过我感觉这样并不好，因为从逻辑意义上，SubComponent 和 AppComponent 并不是一样的东西，只是 SubComponent 使用了一些全局单例变量，所以我个人感觉使用 Component.dependencies 来定义的各个子Component更为恰当一些。

## 以 Component.dependencies 方式

此种方法，可能使用起来更为广泛。

```java
@ActivityScope
@Component(dependencies = AppComponent.class, modules = SplashActivityModule.class)
public interface SplashActivityComponent {
    void inject(SplashActivity activity);
}

@Module
public class SplashActivityModule {
    private SplashActivity splashActivity;

    public SplashActivityModule(SplashActivity splashActivity) {
        this.splashActivity = splashActivity;
    }

    @Provides
    @ActivityScope
    SplashActivity provideSplashActivity() {
        return splashActivity;
    }

    @Provides
    @ActivityScope
    SplashActivityPresenter
    provideSplashActivityPresenter() {
        return new SplashActivityPresenter();
    }
}
```

此种方式是使用 AppComponent 提供的 ApiManager 等进行相应的操作。同样，我们可进行相应的抽象操作：

```java
@ActivityScope
@Component(dependencies = AppComponent.class, modules = { ActivityModule.class, SplashActivityModule.class})
public interface ActivityComponent {

    Activity getActivity();
}
```

这样每个Activity的Component都可以继承 ActivityComponent 来进行相应的注入。

# 关于框架的几点思考

总的来说，关于 Dagger 在 App 中的使用就是上面几点。但是如果使用Dagger 肯定是希望在框架层面上做一些事情的，那么就此延申出来，我们可以思考一些事情：

## 是否有使用 Dagger 的必要

首先第一点，我觉得还是有必要的，首先不用 hard init 有利于解耦。其实，这里的解耦描述并不准确，更多的是将耦合转移。例如你在View 层面 new 一个 Presenter，相当于 hard init Presenter，而如果你是用了 Module的 Provider 样式，则相当于将 View 和 Presenter 的耦合转业到了 Module 和 Presenter，不过仅这样也是有提升的，毕竟 View 中不需要作相应的修改，那里还是以业务逻辑为主，无需关心Presenter 具体怎么变化，只要其接口不会修改就行。还有，我觉得认为没有必要的开发者，可能并没有学会使用Dagger 方法，才导致他们认为没有必要。

## Component 的抽象要怎么做才更合适

第二点，说到 Component ，我们默认使用 Component.dependencies 的形式来进行，并且进行抽象设计：

```java
@ActivityScope
@Component(dependencies = AppComponent.class, modules = ActivityModule.class)
public interface ActivityComponent {

    Activity getActivity();

    void inject(WelcomeActivity welcomeActivity);

}

@Module
public class ActivityModule {
    private Activity mActivity;

    public ActivityModule(Activity activity) {
        this.mActivity = activity;
    }

    @Provides
    @ActivityScope
    public Activity provideActivity() {
        return mActivity;
    }
}
```

这里我们以 [GeekNews](https://github.com/codeestX/GeekNews) 来进行举例，作者抽象出 ActivityComponent ,并定义了一个 BaseActivity 在 onCreate 方法中进行注入。这样我认为是一个好方法，所有继承了 BaseActivity 的Activity 可在初始化时调用 ActivityComponent.inject(this) 方法实现相应的注入，同时不会因为使用 SubComponent 造成每个 Activity 都有唯一的 SubComponent 实例来进行注入，保持了每个 Activity 注入的统一性。同样我们看到这个ActivityComponent 只有唯一的一个 ActivityModule ，而其内部只是提供了同样的 poviderActivity 方法。

其各个 Presenter 都是由构造函数加 @Inject 注解，来生成的依赖，同样在 BaseActivity 直接持有一个相应 Presenter 的引用。我们以其WelcomActivity 为例来看一下，作者思路。

```java
public abstract class BaseActivity<T extends BasePresenter> extends SimpleActivity implements BaseView {

    @Inject
    protected T mPresenter;
    ...

    //在注入成功后将View 手动绑定到 Presenter上
    @Override
    protected void onViewCreated() {
        super.onViewCreated();
        initInject();
        if (mPresenter != null)
            mPresenter.attachView(this);
    }

    //View Destroy时，同时将 Presenter 与 View 解绑
    @Override
    protected void onDestroy() {
        if (mPresenter != null)
            mPresenter.detachView();
        super.onDestroy();
    }
}

//module 不提供给相应的 Provider 函数，由构造函数进行注入
public class WelcomePresenter extends RxPresenter<WelcomeContract.View> implements WelcomeContract.Presenter{
    ...
    @Inject
    public WelcomePresenter(DataManager mDataManager) {
        this.mDataManager = mDataManager;
    }

//持有Presenter 类型通过 泛型传入
public class WelcomeActivity extends BaseActivity<WelcomePresenter> implements WelcomeContract.View {
}
```

作者这样写其实做了个假设：

    每个 View 是和唯一的 Presenter 进行绑定。

针对这个项目没有问题，但是不一定具有通用性。例如一个场景是你可以控制多个对象，同时又可以对每个对象进行单独操作。那么这里一个Presenter 明显是不合适的，一个 Presenter 要对不同对象进行控制，添加，删除，置顶等操作，另一个 Presenter 针对每一个对象进行相应的操作，修改，命令。因此，Presenter 不封装到 BaseActivity 中更为合适。

然后我们看其 ActivityComponent 的 ActivityModule 是一个通用的，不是针对每个 Activty 提供不同的 Presenter，这里其实做了另一个假设：

    每个 View 中只有唯一 Presenter 会需要注入依赖，没有别的依赖需要注入。

针对第二个假设，则更不合适，一个 Activity 中可能有 Fragment Adapter Presenter 以及一些别的依赖需要注入，并非所有的依赖都可以使用 @Inject 修饰构造函数来生成依赖实例，所以我认为上述做法并不友好。

还有一点就是 Presenter 与 View 的绑定时机，我个人认为也是有待商榷的。作者是在注入 Presenter 后，手动将 View attach 到 Presenter 上，这样做确实也没什么问题，但是不符合控制反转思路，因为 Presenter 可以理解是由于 View 主动调用创建的，应该让 Presenter 给一个回调通知 View 说明已经创建完成，可以绑定。因此我觉得将 View 当作参数传入 Presenter，同时进行绑定更为合理一些。

```java
// 所有 Presenter 的实现基类
public abstract class BasePresenter<T extends BaseContract.View>
    implements BaseContract.Presenter {

  final protected ApiClient mApiClient;
  final protected AccountMgr mAccountMgr;

  ...
  public BasePresenter(ApiClient apiClient, AccountMgr accountMgr, T view) {
    mApiClient = apiClient;
    mAccountMgr = accountMgr;
    mView = view; //创建的同时直接绑定
  }

  @Override
  public void onDestroy() {
    this.mView = null; //View 在 Destroy 时进行解绑
  }
}

//SplashModule 提供 SplashPresenter
@Module
public class SplashModule {

  @Provides
  @ActivityScope
  public SplashContract.Presenter providePresenter(ApiClient apiClient, BaseContract.View view, AccountMgr accountMgr) {
    if (view instanceof SplashContract.View)
      return new SplashPresenter(apiClient, (SplashContract.View)view, accountMgr);
    return null;
  }

}

public class SplashActivity extends BaseInjectActivity implements SplashContract.View {

  @Inject SplashContract.Presenter mPresenter;

  // 通用注入，注入完成后 开始操作 Presenter
  @Override protected void onViewCreated(Bundle savedInstanceState) {
    super.onViewCreated(savedInstanceState);
    mActivityComponent.inject(this);
    mPresenter.start();
    }

  // onDestroy 时进行解绑
  @Override protected void onDestroy() {
    super.onDestroy();
    mPresenter.onDestroy();
  }
}

//Presenter 的实现类
public class SplashPresenter extends BasePresenter<SplashContract.View>
    implements SplashContract.Presenter {


  public SplashPresenter(ApiClient apiClient, SplashContract.View view, AccountMgr accountMgr) {
    super(apiClient,accountMgr, view);
  }
}
```

主要就是改了两个部分：1. Presenter 的注入从基类提出；2. 将 View 作为参数传入 Presenter 的构造函数中，实现创建即绑定。

## Module 中放入哪些实现

针对 ActivityModule （这里是泛指每个 Activity 的 Module，非抽象出的 ActivityModlue）应该提供哪些实现，这个应该是根据不同的情况进行区别，这里给出几个例子。

```java
@Module public class ActivityModule {
  private Activity mActivity;

  public ActivityModule(Activity activity) {
    this.mActivity = activity;
  }

  //提供 RxPermissions
  @Provides
  @ActivityScope
  public RxPermissions provideRxPermission() {
    return new RxPermissions(mActivity);
  }

  //提供 Presenter
  @Provides
  @ActivityScope
  public Presenter providePresenter(BaseContract.View view) {
      return new ImplPresenter(view)
  }

  //提供 Adapter
  @Provides
  @ActivityScope
  public Adapter povideChildAdapter(Activity activity) {
    Adapter adapter = new Adapter();
    return adapter;
  }

  //提供 传入参数
  @Provides
  @ActivityScoped
  public String provideTaskId() {
    return mActivity.getIntent().getStringExtra(EXTRA_STR);
  }
}
```

当然上面的写到比较简略，不过就是这个意思。

# 后记

总体来说 Dagger 的学习成本较高，但是使用熟练的话，解耦的效果还是不错的，同时使 App 的业务逻辑更为清晰，不需要一堆构造函数。我个人比较推荐将依赖的生成都提供在Module中，哪怕是最简单 new 一个变量，因为更清楚同时也不会修改依赖类。