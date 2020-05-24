---
date : 2018-09-24T15:21:57+08:00
tags : ["View", "Android"]
title : "View 的一些整理 嵌套滑动的解决方案"

---

之前我们讲过了嵌套滑动机制，上一篇也分析了事件分发的流程，那么这一篇我们结合之前的内容来分析下两种机制实现嵌套滑动怎样做。

### 准备阶段
我们这次打算通过自定义两个控件的方式，来实现嵌套滑动机制的分析。最终实现类似于 ViewPager 结合 RecyclerView 的常见的控件组合。其中 ViewPager 可以左右滑动，Recycler 可以上下滑动。

```java
// 自定义的父控件
public class ParentViewInter extends ViewGroup {

  public ParentViewInter(Context context) {
    this(context, null);
  }

  public ParentViewInter(Context context, AttributeSet attrs) {
    this(context, attrs, 0);
  }

  public ParentViewInter(Context context, AttributeSet attrs, int defStyleAttr) {
    super(context, attrs, defStyleAttr);
  }

  @Override
  protected void onLayout(boolean changed, int l, int t, int r, int b) {
    int count =getChildCount();
    for (int i = 0; i < count; i++) {
      View child = getChildAt(i);
      child.layout(l+i*getMeasuredWidth(), t, r+i*getMeasuredWidth(), b);
    }
  }

}

// 自定义的子控件
public class ChildViewInter extends AppCompatTextView {

  public ChildViewInter(Context context) {
    this(context,null);
  }

  public ChildViewInter(Context context, AttributeSet attrs) {
    this(context, attrs,0);
  }

  public ChildViewInter(Context context, AttributeSet attrs, int defStyleAttr) {
    super(context, attrs, defStyleAttr);
  }

}
```
这里我们父控件使其继承于 ViewGroup ，因此需要重写 onLayout 方法；同时，子控件为了方便观察上下滑动的效果，我们使其继承于 TextView 这样上下滑动时字的内容可以作为参照物。

### 事件拦截分发
#### 流程分析
上篇中我们讲过，通过拦截事件分发主要是通过3个函数实现的：

    dispatchTouchEvent
    onInterceptTouchEvent
    onTouchEvent
通常，dispatchTouchEvent 不会完全重写，因为它涉及向子控件进行事件分发。这里我们主要考虑后面两个函数。我们的思路是这的：

    针对 onInterceptTouchEvent 函数：
      ACTION_DOWN 事件父控件不会也不应该拦截否则后面的事件都不会传入到子控件里了；
      ACTION_MOVE 事件父控件判断如果是左右滑动便拦截下来，自己来处理。

    针对 onTouchEvent 函数：
      ACTION_DOWN 事件父子控件都要返回 true 表示自己可以处理，不然后续事件传不过来；
      ACTION_MOVE 事件则无所谓父子控件是否拦截，因为总会传过来的。

因此流程也就清晰了。

#### ParentViewInter 
这里我们先重写 ParentViewInter 里的方法。
```java
public boolean onInterceptTouchEvent(MotionEvent ev) {
  boolean result = false;
  switch (ev.getAction()){
    case MotionEvent.ACTION_DOWN:
      mLastX = (int)(ev.getX()+0.5);
      mLastY = (int)(ev.getY()+0.5);
      break;
    case MotionEvent.ACTION_MOVE:
      result = (Math.abs(ev.getX() - mLastX) > Math.abs(ev.getY() - mLastY));
      break;
  }
  return result;
}
```
这里滑动的判断，我们之前在 [View 的一些整理 参考系与滑动](/post/View 的一些整理 参考系与滑动) 介绍过，简单的说就是每次滑动后都和 ACTION_DOWN 的坐标进行对比，判断横向的滑动是否大于纵向的滑动，如果是的话拦截，反之则不拦截。
```java
public boolean onTouchEvent(MotionEvent event) {
  boolean result = false;
  switch (event.getAction()){
    case MotionEvent.ACTION_DOWN:
      result = true;
      break;
    case MotionEvent.ACTION_MOVE:
      scrollBy(-(int)(event.getX()-mLastX),0);
      break;
  }
  mLastX = (int)(event.getX()+0.5);
  mLastY = (int)(event.getY()+0.5);

  return result;
}
```
如上说的，ACTION_DOWN 要拦截，不然后续事件父控件收不到；同时在收到 ACTION_MOVE 进行下内容滑动，左右移动到相应的位置。移动完更新下最近点击的相应的位置即可。

#### ChildViewInter
相比于 ParentViewInter 会更简单一些，因为只需要重写 onTouchEvent 函数。
```java
public boolean onTouchEvent(MotionEvent event) {
  boolean result = false;
  switch (event.getAction()){
    case MotionEvent.ACTION_DOWN:
      result = true;
      break;
    case MotionEvent.ACTION_MOVE:
      scrollBy(0,-(int)(event.getY()-mLastY));
      break;
  }
  mLastX = (int)(event.getX()+0.5);
  mLastY = (int)(event.getY()+0.5);

  return result;
}
```
内容和 ParentViewInter 的类似不过是上下移动的。最后我们看下实际效果：

![事件拦截](view_intercepttouch.gif)

这里面有些问题例如正在上下滑动时，突然左右滑动，那么无法再上下滑动，这是由于左右滑动时父控件拦截了事件，而此时已从子控件 onTouchEvent 出来，无法再次处理上下滑动事件，因此会显得体验很差。这个问题可以在 ParentViewInter.onInterceptTouchEvent 增加判断，如果在一次整体操作的过程中出现了上下滑动，则左右滑动不再拦截直到事件序列结束。这里就不再演示。

### 自定义 NestedScrolling
除了上述拦截事件，我们还可以重写 NestedScrolling 的相关方法来实现嵌套滑动。这里我直接把代码贴出来，我们来看下有什么不同：
```java
public class ParentViewNested extends ViewGroup  {
  
  //省略了构造函数
  ...

  @Override
  protected void onLayout(boolean changed, int l, int t, int r, int b) {
    int count = getChildCount();
    for (int i = 0; i < count; i++) {
      View child = getChildAt(i);
      child.layout(l + i * getMeasuredWidth(), t, r + i * getMeasuredWidth(), b);
    }
  }

  @Override
  public boolean onStartNestedScroll(View child, View target, int nestedScrollAxes) {
    return target instanceof ChildViewNested;
  }

  @Override
  public void onNestedPreScroll(View target, int dx, int dy, int[] consumed) {
    if (Math.abs(dy) > Math.abs(dx)) {
      consumed[0] = dx;
    } else {
      scrollBy(-dx, 0);
    }
  }
}


public class ChildViewNested extends AppCompatTextView  {
  private NestedScrollingChildHelper mChildHelper;

  int mLastX = -1;
  int mLastY = -1;
  int[] consumed = new int[2];
  int[] offsets = new int[2];

  //省略了部分构造函数
  ...
  public ChildViewNested(Context context, AttributeSet attrs, int defStyleAttr) {
    super(context, attrs, defStyleAttr);
    mChildHelper = new NestedScrollingChildHelper(this);
    mChildHelper.setNestedScrollingEnabled(true);
  }

  @Override
  public boolean onTouchEvent(MotionEvent event) {
    boolean result = true;
    switch (event.getAction()) {
      case MotionEvent.ACTION_MOVE:
        int deltaX = (int) (event.getRawX() + 0.5) - mLastX;
        int deltaY = (int) (event.getRawY() + 0.5) - mLastY;

        mChildHelper.startNestedScroll(
            Math.abs(deltaX) > Math.abs(deltaY) ? ViewCompat.SCROLL_AXIS_HORIZONTAL
                : ViewCompat.SCROLL_AXIS_VERTICAL);

        if(mChildHelper.dispatchNestedPreScroll(deltaX, deltaY, consumed, offsets))
          scrollBy(0,-(deltaY-consumed[1]));
    }

    mLastX = (int) (event.getRawX() + 0.5);
    mLastY = (int) (event.getRawY() + 0.5);
    return true;
  }
}
```
对于 ParentViewNested 没有重写事件分发函数，只是重写了 onStartNestedScroll 与 onNestedPreScroll 函数。这两个函数分别是开启嵌套滑动和预滑动操作。而 ChildViewNested 则是类似于 ChildViewInter ，仅重写了 onTouchEvent ，不同的是在 ACTION_MOVE 事件中，调用了 dispatchNestedPreScroll 即向上询问是否要消耗对应向的平移。

在 ParentViewNested.onNestedPreScroll 方法中如果 x 轴向位移大于 y 轴位移，则将内容进行 -dx 距离的平移，否则消耗 x 轴向的位移。同时当消耗 x 向位移时，dispatchNestedPreScroll 会返回 true 表示，父控件有消耗，子控件可以继续消耗 y 向位移，因此子控件可以进行 y 轴向的内容平移。这里有一个小点需要注意，就是 ChildViewNested.onTouchEvent 获取点击位置的方式需要用 getRawX(),getRawY() 因为父控件的内容会发生改变，因此 getX() getY() 可能是会发生改变的，会出现滑动抖动的现象。

### 小结
总体来说，在解决嵌套时，两种思路完全基本不一样。第一种事件拦截，主要是要父控件知道什么时候该进行拦截，如果拦截了则自己要做什么，不拦截则放给子控件。第二种嵌套滑动，主要是子控件在进行滑动之前先询问父控件是否要进行滑动，父控件只考虑自己要不要消耗就好，完全不用管拦截的事儿，父控件消耗之后子控件再进行消耗就好了。前者是一个自上而下的过程，后者则更像自下而上，读者可以自行体会。


