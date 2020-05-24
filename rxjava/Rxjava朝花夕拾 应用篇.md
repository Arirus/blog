---
date : 2018-08-13T16:55:06+08:00
tags : ["Rxjava", "Android", "用法"]
title : "Rxjava朝花夕拾 应用篇"

---

本次源码分析是以 GitHub 上 RxJava 仓库 [2.1.1](https://github.com/ReactiveX/RxJava)版本来进行的分析，同时以[官方中文文档](https://legacy.gitbook.com/book/mcxiaoke/rxdocs/details)作为参考。

上一篇中，我们分析对比了同步和异步情况下，背压的原理，并针对各种情况，分析了不同的解决办法，总而言之一句话就是“不让上游的漏斗中的油流出来”。好了说了这么多，我们这篇来分享几个 Android 开发中的常见用法。这次对于过于简单的例子,我们就不详讲了,如线程切换用法(毕竟这是Rxjava的基础功能,我们还是在Rx上做文章较多).

<!--more-->

# 对于集合的遍历操作

由于 Android 低版本(API 24 以下)不支持 Java8 的流式操作，因此我们可以使用 Rxjava 将其转化为支持流式操作。

```Java
List<String> list = Arrays.asList("Arirus","is","me","Welcome","to","my","blog");
String pattern="^[A-Z]+\\w{6}";

for (String content: list) {
  if (Pattern.matches(pattern,content)) Log.i(ARIRUS, "onCreate: "+content);
}

Observable.fromIterable(list).filter(content->Pattern.matches(pattern,content)).blockingForEach(
    s -> Log.i(ARIRUS, "accept: "+s));

list.stream()
    .filter(content -> Pattern.matches(pattern, content))
    .forEach(content -> Log.i(ARIRUS, "accept: " + content));
```

这里给出三个对比：

第一个是常规写法，对集合进行遍历，对于每一个遍历的对象进行判断进而进行响应的操作；第二个是使用 Rxjava 转换后，从一个集合中生成一个 Observable，将一个个的集合元素封装成元素发射到下游，同时进行过滤和 forEach 操作。最后注意要 blocking 一下转换成阻塞操作;第三个则是Java8支持的Stream操作，基本原理同上，字面意思更好理解。

因此，再不支持Java8的情况下，使用 Observable 转换来支持流操作是个不错的选择。

# 与 UI 结合使用

我觉得使用 RX 来进行 UI 操作是一种全新的体验。现在我们考虑一种情景：响应点击 Button 的点击事件，同时要求每秒内只能响应第一次，也就是我们常说的消抖功能，使用 RX 我们可以这么写：

```Java
private static class Ding {
  Date time;
  public Ding() {
    time = new Date(System.currentTimeMillis());
  }
}

Flowable.just(button)
    .flatMap((Function<Button, Publisher<Ding>>) button1 ->
        Flowable.create(emitter -> button1.setOnClickListener(v -> emitter.onNext(new Ding())),
        BackpressureStrategy.ERROR))
    .throttleFirst(1, TimeUnit.SECONDS)
    .subscribe(ding -> Log.i(ARIRUS, "onCreate: " + ding.time));

//这里我们使用了，Flowable，当然也可以使用 Observable
```

这里开始对一个 Button 进行观察，通过 `flatMap` 使用将一个 Button 流，转换成了Button的点击事件流，同时对于流进行了限制，每秒只能有1个点击事件传输到下游。一个链式的调用很舒服。

再另举一个例子：登录界面，要求两个 EditText 都填上了相应的内容才可以点击登录按钮。如果使用 RX 我们可以这么写：

```Java
Flowable<CharSequence> flowable1 = Flowable.just(ed1)
    .flatMap(editText -> Flowable.create((FlowableOnSubscribe<CharSequence>) emitter -> {
      editText.addTextChangedListener(new TextWatcher() {
        @Override
        public void onTextChanged(CharSequence s, int start, int before, int count) {
          emitter.onNext(s);
        }
        ...
      });
      emitter.onNext("");
    }, BackpressureStrategy.ERROR));

Flowable<CharSequence> flowable2 = Flowable.just(ed2)
    .flatMap(editText -> Flowable.create((FlowableOnSubscribe<CharSequence>) emitter -> {
      editText.addTextChangedListener(new TextWatcher() {

        @Override
        public void onTextChanged(CharSequence s, int start, int before, int count) {
          emitter.onNext(s);
        }
        ...
      });
      emitter.onNext("");
    }, BackpressureStrategy.ERROR));

Flowable.combineLatest(flowable1, flowable2,
            (charSequence, charSequence2) -> charSequence.length()> 0
                    && charSequence.length() < 20
                    && charSequence2.length() > 0
                    && charSequence2.length() < 10).subscribe(button::setEnabled);
```

这里面简略了 TextWatcher 的方法，思路还是同上， 将每个 EditText 的观察转换为其“内容变化”的观察，同时对于观察的数据进行判断，符合规定的话，才会做响应的操作。
类似的操作还有很多，这里就不一一举例了，可以使用 RxBinding 库来进行更为细致的操作。

# 操作数据源

通常对于某些情况，我们的数据源来源不是一个。例如，淘宝App，首页资讯在断网打开的情况下，依然会显示，因为其在本地对于图片等信息有缓存，因此有时会看到界面一闪的刷新一下，这就是来网络数据来的对界面进行了刷新。
我们可以使用 `merge` 方法来将不同的数据源进行合并。

```java
Observable<String> observable1 =
    Observable.interval(1, TimeUnit.SECONDS).take(5).map(aLong -> "资源1:"+aLong);
Observable<String> observable2 =
    Observable.interval(2,1, TimeUnit.SECONDS).take(5).map(aLong -> "资源2:"+aLong);
Observable<String> observable3 =
    Observable.interval(4,1, TimeUnit.SECONDS).take(5).map(aLong -> "资源3:"+aLong);

//将不同源的数据合并并打印出来
Observable.merge(observable1, observable2, observable3)
    .subscribe(string -> Log.i(ARIRUS, "subscribe: " + string));

Log:
I/ARIRUS: subscribe: 资源1:0
I/ARIRUS: subscribe: 资源1:1
I/ARIRUS: subscribe: 资源2:0
I/ARIRUS: subscribe: 资源1:2
I/ARIRUS: subscribe: 资源2:1
I/ARIRUS: subscribe: 资源1:3
I/ARIRUS: subscribe: 资源2:2
I/ARIRUS: subscribe: 资源3:0
I/ARIRUS: subscribe: 资源1:4
I/ARIRUS: subscribe: 资源2:3
I/ARIRUS: subscribe: 资源3:1
I/ARIRUS: subscribe: 资源2:4
I/ARIRUS: subscribe: 资源3:2
I/ARIRUS: subscribe: 资源3:3
I/ARIRUS: subscribe: 资源3:4
// 这里便将三个不同的数据源数据合并起来了。并将三个数据源的数据打印出来。

//对于某些情况，我们想要默认显示源1的数据，如果源2的数据来了，则显示后者的，可以结合 takeUntil 一起使用。这种情况，在某些咨询类页面较为常见。
Observable.merge(observable1.takeUntil(observable2), observable2.takeUntil(observable3), observable3)
    .subscribe(string -> Log.i(ARIRUS, "subscribe: " + string));
Log：
I/ARIRUS: subscribe: 资源1:0
I/ARIRUS: subscribe: 资源2:0
I/ARIRUS: subscribe: 资源2:1
I/ARIRUS: subscribe: 资源3:0
I/ARIRUS: subscribe: 资源3:1
I/ARIRUS: subscribe: 资源3:2
I/ARIRUS: subscribe: 资源3:3
I/ARIRUS: subscribe: 资源3:4
//这样在第一个发射的过程中，如果发现源2开始发射数据，则前者停止发射，同理源3

//当然也可以结合 take() 来取钱几个内容，起到一个优先级的作用，这里就不详细讲了
```

# 与 `RecyclerView` 的结合使用

RecyclerView 应该是最常用的控件之一，应为手机设备的特性，所以经常使用 `RecyclerView` 来进行浏览信息。使用 RecyclerView 有两个需要注意的地方：

    1.如何添加数据；
    2.如何响应 itemCell 的点击事件
由于通常显示数据的时候，不会一下显示完，而是一部分一部分的添加显示（微博），同时 RecyclerView 没有itemCell的点击事件监听。因此 RecyclerView 使用的时候通常需要二次封装。那么常规的解决方式应该是：

    1.监听 RecyclerView 滑动事件；
    2.设置 Adapter 的 ClickListener
如果这里我们将 Rxjava 与 RecyclerView 进行结合，那么可以更清晰的方式来解决上述问题。

```Java
//调用处进行响应的订阅，用来接收通知事件等。
mCustomAdapter.clickEvent.subscribe(integer -> Toast.makeText(this, "点击item："+integer, Toast.LENGTH_SHORT)
    .show());
mCustomAdapter.updateEvent.subscribe(updateEvent -> mPresenter.update(mPage++));

private static class CustomAdapter extends RecyclerView.Adapter<ItemView> {

  private List<Integer> mList;
  private PublishProcessor<Integer> mProcessorClick; //接收或者发射 点击 itemCell的 position
  private PublishProcessor<UpdateEvent> mProcessorUpdate; //接收或者发射 更新 list的通知
  public Flowable<Integer> clickEvent ; //暴露对象
  public Flowable<UpdateEvent> updateEvent; //暴露对象

  boolean isLoading; //更新标志位

  public void setLoading(boolean loading) {
    isLoading = loading;
  }

  public CustomAdapter() {
    mList =new ArrayList<>();
    mProcessorClick = PublishProcessor.create();
    mProcessorUpdate : PublishProcessor.create();
    clickEvent = mProcessorClick;
    updateEvent = mProcessorUpdate;
  }

  @Override
  public ItemView onCreateViewHolder(ViewGroup parent, int viewType) {
    View view = LayoutInflater.from(parent.getContext())
        .inflate(android.R.layout.simple_list_item_activated_1, parent, false);
    return new ItemView(view);
  }

  @Override
  public void onBindViewHolder(ItemView holder, int position) {
    holder.bind(mList.get(position) , mProcessorClick);
    if (position+2>=getItemCount() && !isLoading){
      mProcessorUpdate.onNext(UpdateEvent.INSTANCE);
      isLoading = true;
    }
  }

  @Override
  public int getItemCount() {
    return mList.size();
  }

  enum  UpdateEvent {
    INSTANCE
  }
}

private static class ItemView extends RecyclerView.ViewHolder {

  private final TextView mTextView;
  public ItemView(View itemView) {
    super(itemView);
    mTextView = itemView.findViewById(android.R.id.text1);
  }
  public void bind(Integer integer,PublishProcessor<Integer> processor) {
    mTextView.setText(String.valueOf(integer));
    mTextView.setOnClickListener(v-> processor.onNext(integer));
  }
}
```

PublishProcessor 在之前我们说过，其类似与 PublishSubject 既可以接收事件也可以发送事件，这里我们便利用了这一特性。当 itemCell 被点击的时候，或者当 RecyclerView 需要添加新的数据的时候，便会向 PublishProcessor 发射通知，而在外部订阅出会收到通知，从而进行响应的操作。这样将所有绑定操作都分配到 Adapter 和 itemCell 的内部，外部只需要暴露出一个 Observable，外部订阅后便会都到通知。

# RxBus 初探

事件总线等相关概念我们在这里不再重复强调，这里只是来介绍 RxBus ,看看后期有没有可能写一篇 RxBus 与 EventBus 对比的文章了（挖坑）。
这里我们给 RxBus 提几个要求：

    1.满足事件发布，订阅的需求
    2.针对同类型事件，支持已经不同标志位进行订阅
    3.支持 Sticky 事件

针对第一点，上面的 RecyclerView 其实已经为我们指出使用方法了，使用 PublishProcessor ，可以在不同地方进行发布和订阅。注意由于 RxBus 可以在多线程中使用因此，使用 FlowableProcessor 更好一些，因为后者是线程安全的，可以保证事件前后发送和接收顺序一致性。第二点，我们需要对同一类型的事件，进行区分，因此在发布时我们需要加上一个 TAG 标志位，同时根据 TAG 进行接收时的过滤。第三点，由于 PublishProcessor 的特性（不会保留之前的事件，只会发送订阅之后的发布的事件），因此另需要一个 HashMap 来存储(BehaviorProcessor 也不行，因为我们是将所有的事件都发送到同一个 Processor 中，因此无法保证前一个和后一个事件是同一类型的)。因此我们可以向如下这样写。

```Java
public enum RxBus {

  INSTANCE;

  FlowableProcessor<Object> mFlowableProcessor;
  ConcurrentHashMap<Object, CompositeDisposable> mDisposableMap;
  ConcurrentHashMap<Class, Event> mStickyMap;

  RxBus() {
    mFlowableProcessor = PublishProcessor.create().toSerialized();
  }

  public void post(Object o, String tag, boolean isSticky) {
    Event event = new Event(o, tag);
    mFlowableProcessor.onNext(event);
    if (isSticky) addStickyMap(event);
  }

  public <T> Disposable doSubscribe(Class<T> type, String tag,boolean isSticky, Consumer<T> consumer) {
    Flowable<Event> flowable = mFlowableProcessor.ofType(Event.class);

    if (isSticky && mStickyMap.get(type)!=null)
      return Flowable.merge(Flowable.just(mStickyMap.get(type)),flowable)
          .filter(event -> event.mObject.getClass() == type && tag.equals(event.mTag))
          .map(event -> (T)event.mObject).subscribe(o->consumer.accept((T)o));

    return flowable
        .filter(event -> event.mObject.getClass() == type && tag.equals(event.mTag))
        .map(event -> (T)event.mObject)
        .subscribe(o -> consumer.accept(o));
  }
  ...
  private static class Event {
    Object mObject;
    String mTag;

    public Event(Object object, String tag) {
      mObject = object;
      mTag = tag;
    }
  }}

}
```

这里简写了部分内容，只保留了发布和订阅部分。逻辑其实和上面的 RecyclerView 封装是一样的，使用一个 FlowableProcessor 来发布和订阅事件，同时使用一个 HashMap 保存 Sticky 事件实例，订阅时，如果此事件实例存在就和 FlowableProcessor 源一起订阅，否则只订阅后者，当然了这里我对于事件匹配的判读都是最直接方式进行的书写，没有进行封装。调用则可以采用如下方式调用

```Java
RxBus.INSTANCE.post("Click Post", "Click", false); //发送事件
RxBus.INSTANCE.doSubscribe(String.class, "Click", true, s -> textView.setText(s)) //接收事件
```

# 小结

本篇中，我们把 Android 的相关操作和 Rxjava 进行了简单的结合。其实站在 Rxjava 的角度，就只有两类，一类是冷 Observable，另一类是热 Observable ，针对不同的情况灵活使用使得代码更为灵活。
