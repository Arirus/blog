---
date : 2018-09-11T17:08:39+08:00
tags : ["嵌套滑动","自定义", "Android", "用法"]
title : "View 的一些整理 从 NestedScrolling 到 Behavior"

---

# 写在最前面
这次的知识点应该是很久之前的知识点了，嵌套滑动相关，不过之前竟然没有认真总结一下，那就正好这次好好写一篇文章，把这里梳理一下。本文采用的源码均来自 API 26 的官方源码。本文会涉及到几个常用的 Material Design 控件，假设读者知道，并会大致使用。本篇结合《嵌套滑动的解决方案》一起看，效果更好。

# NestedScrolling 机制
滑动的源远流长咱们就不说了，这次我们来说嵌套滑动。站在之前的 `View 事件分发` 的角度来说，嵌套滑动是很神奇的，毕竟如果一个动作被父控件拦截了，子控件是没法继续触发的。但是 NestedScrolling 却给了一个很好的例子，怎样让父子控件关于该谁滑动进行沟通。包导入也不说了，毕竟这些大家应该都懂，我们直接来说说 NestedScrolling 机制。
## 相关接口与类
NestedScrolling 机制中，有4个相关接口和类：

    NestedScrollingChild //子控件需要实现的接口
    NestedScrollingParent //父控件需要实现的接口

    NestedScrollingChildHelper //子控件辅助类
    NestedScrollingParentHelper //父控件辅助类

我们来对比下父子控件需要实现接口的方法：

|   NestedScrollingChild |  NestedScrollingParent | 说明 |
|   ---- |----| --- |
|   void setNestedScrollingEnabled(boolean enabled);<br> boolean isNestedScrollingEnabled(); ||子 View 设置/是否支持嵌套滑动|
|  boolean startNestedScroll(int axes);<br> void stopNestedScroll(); | boolean onStartNestedScroll(View child, View target, int nestedScrollAxes);<br> public void onNestedScrollAccepted(View child, View target, int nestedScrollAxes); <br> public void onStopNestedScroll(View target); | 父子控件开始/停止支持嵌套滑动<br>axes 的值代表方向|
| boolean hasNestedScrollingParent();||子控件当前是否有支持滑动嵌套的父控件|
| boolean dispatchNestedPreScroll(int dx, int dy, int[] consumed, int[] offsetInWindow);<br>boolean dispatchNestedScroll(int dxConsumed, int dyConsumed, int dxUnconsumed, int dyUnconsumed, int[] offsetInWindow);| void onNestedPreScroll(View target, int dx, int dy, int[] consumed); <br> void onNestedScroll(View target, int dxConsumed, int dyConsumed, int dxUnconsumed, int dyUnconsumed); | 嵌套滑动相关函数|
|boolean dispatchNestedFling(float velocityX, float velocityY, boolean consumed);<br> boolean dispatchNestedPreFling(float velocityX, float velocityY);|boolean onNestedPreFling(View target, float velocityX, float velocityY);<br> boolean onNestedFling(View target, float velocityX, float velocityY, boolean consumed);|惯性滑动相关函数|

可以看出来，NestedScrollingChild 与 NestedScrollingParent 的函数大部分是一一对应的。有几个方法，我们要详细说明一下：
`boolean startNestedScroll(int axes)` 中的参数表示滑动方向，有两个可选的值：SCROLL_AXIS_HORIZONTAL 和 SCROLL_AXIS_VERTICAL。
```java
int nestedScrollAxis = ViewCompat.SCROLL_AXIS_NONE;
if (canScrollHorizontally) {
    nestedScrollAxis |= ViewCompat.SCROLL_AXIS_HORIZONTAL;
}
if (canScrollVertically) {
    nestedScrollAxis |= ViewCompat.SCROLL_AXIS_VERTICAL;
}
startNestedScroll(nestedScrollAxis);
```
上述代码是 RecyclerView 中对于嵌套滑动的设置。
`boolean dispatchNestedPreScroll(int dx, int dy, int[] consumed, int[] offsetInWindow);` 这个函数f发生在子控件滑动之前，给嵌套的父控件一个消费滑动事件的机会，这个也就是子控件对于父控件的`沟通`。如果返回了 true ，则说明父控件接受这个滑动询问，进行了滑动，`dx` `dy` 分别表示本次滑动的像素，`consumed` 输出父控件总共消费的距离，`offsetInWindow`输出子控件在此过程位置的偏移量。这里我们可以看看 RecyclerView 关于这里的使用：
```java
case MotionEvent.ACTION_MOVE: {
    ....
    if (dispatchNestedPreScroll(dx, dy, mScrollConsumed, mScrollOffset)) {
        dx -= mScrollConsumed[0];
        dy -= mScrollConsumed[1];
        vtev.offsetLocation(mScrollOffset[0], mScrollOffset[1]);
        // Updated the nested offsets
        mNestedOffsets[0] += mScrollOffset[0];
        mNestedOffsets[1] += mScrollOffset[1];
    }
  ...
}
```
在 onTouchEvent 中的滑动操作，询问父控件是否要消费该事件，如果是的话，会对剩余的值进行处理。`boolean dispatchNestedScroll(int dxConsumed, int dyConsumed, int dxUnconsumed, int dyUnconsumed, int[] offsetInWindow);` 子控件在消费完事件后，询问父控件是否还要进行余下的滑动，来消费 `Unconsumed`。为输出参数，offsetInWindow 用于子view获取父view位置的偏移量。同样我们可以看下 RecyclerView 关于此处的使用：
```java
boolean scrollByInternal(int x, int y, MotionEvent ev) {
  ...
  if (dispatchNestedScroll(consumedX, consumedY, unconsumedX, unconsumedY, mScrollOffset)) {
      // Update the last touch co-ords, taking any scroll offset into account
      mLastTouchX -= mScrollOffset[0];
      mLastTouchY -= mScrollOffset[1];
      if (ev != null) {
          ev.offsetLocation(mScrollOffset[0], mScrollOffset[1]);
      }
      mNestedOffsets[0] += mScrollOffset[0];
      mNestedOffsets[1] += mScrollOffset[1];
  }
  ...
}
```

NestedScrollingParent 中的函数大部分是和 NestedScrollingChild 中的一一对应，这里说一点的就是：`child` 与 `target` 的区别，前者是父控件的直接子控件里面包括了后者，后者是实现了 NestedScrollingChild 的子控件。所以对于 NestedScrolling 机制，不一定是要求 NestedScrollingParent 直接子控件实现了 NestedScrollingChild 接口，允许非直接子控件。

## NestedScrollingChildHelper 与 NestedScrollingParentHelper
上面我们分析了 NestedScrollingChild 与 NestedScrollingParent 的相关函数，那究竟要怎么实现里面的函数呢？Google 提供了两个 helper 类，来协助我们完成请求传递等任务。
```java
public class NestedScrollingChildHelper {
    private final View mView;
    private ViewParent mNestedScrollingParent;
    private boolean mIsNestedScrollingEnabled;
    private int[] mTempNestedScrollConsumed;

  public NestedScrollingChildHelper(View view) {
        mView = view;
    }
  ... //源码较多就不全部贴出，可以自己翻看
}
```
NestedScrollingChildHelper 实现了 NestedScrollingChild 全部的方法，因此当我们在使用 NestedScrollingChild 时可以直接全部调用 helper 来进行实现。例如，startNestedScroll 就是向上遍历看看有没有合适的 Parent 支持嵌套滑动；dispatchNestedScroll 里则是间接调用 NestedScrollingParent 的 onNestedScroll 的方法。

```java
public class NestedScrollingParentHelper {
    private final ViewGroup mViewGroup;
    private int mNestedScrollAxes;

    public NestedScrollingParentHelper(ViewGroup viewGroup) {
        mViewGroup = viewGroup;
    }

    public void onNestedScrollAccepted(View child, View target, int axes) {
        mNestedScrollAxes = axes;
    }

    public int getNestedScrollAxes() {
        return mNestedScrollAxes;
    }

    public void onStopNestedScroll(View target) {
        mNestedScrollAxes = 0;
    }
```
NestedScrollingParentHelper 实现的方法只有这3个，因此如果想要实现嵌套滑动，还需要自己实现 onNestedScroll 等方法，因此后者的实现将是我们后面要重要讨论的内容。

# NestedScrolling 实现

## View 与 ViewGroup 的默认实现
在 API 21 之后，View 默认实现了 NestedScrollingChild 方法，ViewGroup 默认实现了 NestedScrollingParent 的方法，所以可以认为 API 21 之后的控件都是默认支持嵌套滑动的。因此如果我们自定义控件的时候 mini 是21以上，可以完全不使用 helper 类，直接重写 view 中的相应方法即可。我们先来看看其内部代码是怎样实现的：
```java
    @Override
    public boolean onStartNestedScroll(View child, View target, int nestedScrollAxes) {
        return false;
    }

    @Override
    public void onNestedScrollAccepted(View child, View target, int axes) {
        mNestedScrollAxes = axes;
    }

    @Override
    public void onStopNestedScroll(View child) {
        // Stop any recursive nested scrolling.
        stopNestedScroll();
        mNestedScrollAxes = 0;
    }

    @Override
    public void onNestedScroll(View target, int dxConsumed, int dyConsumed,
            int dxUnconsumed, int dyUnconsumed) {
        // Re-dispatch up the tree by default
        dispatchNestedScroll(dxConsumed, dyConsumed, dxUnconsumed, dyUnconsumed, null);
    }

    @Override
    public void onNestedPreScroll(View target, int dx, int dy, int[] consumed) {
        // Re-dispatch up the tree by default
        dispatchNestedPreScroll(dx, dy, consumed, null);
    }

    @Override
    public boolean onNestedFling(View target, float velocityX, float velocityY, boolean consumed) {
        // Re-dispatch up the tree by default
        return dispatchNestedFling(velocityX, velocityY, consumed);
    }

    @Override
    public boolean onNestedPreFling(View target, float velocityX, float velocityY) {
        // Re-dispatch up the tree by default
        return dispatchNestedPreFling(velocityX, velocityY);
    }

    public int getNestedScrollAxes() {
        return mNestedScrollAxes;
    }
```
ViewGroup 作为 NestedScrollingParent 中默认是没有开启嵌套滑动，onStartNestedScroll 直接返回了 false。同时 onNestedScroll 等方法都是直接向上一层继续询问有没有接受嵌套滑动的控件，它本身只是起到了一个传递的作用，因此一个 ViewGroup 本身没有开启嵌套滑动的 Parent 功能，只是简单的将事件向上层传递。因此自己如果实现一个 NestedScrollingParent 一定要实现上面的5个方法。


相比于 ViewGroup View，中则实现了全部的 NestedScrollingChild 的方法，同时也有了详细的实现。实际上通过对比发现，二者在实现上没有太大差别，几乎一摸一样。所以如果我们实现一个自定义的 NestedScrollingChild，直接继承于 View 是可以直接使用的，当然这是建立在最小支持版本大于等于21，如果小于，还是需要使用 Helper 类进行支持的。

## 自定义实现
本节我们来实现一对儿简单的嵌套滑动父子控件。
```java
public class NestedScrollingChildView extends AppCompatTextView {
  private int lastY;
  private final int[] offset = new int[2];
  private final int[] consumed = new int[2];

    ... //省略构造函数
  @TargetApi(21)
  @Override
  public boolean onTouchEvent(MotionEvent event) {
    switch (event.getAction()){
      case MotionEvent.ACTION_DOWN:
        lastY = (int) event.getRawY();
        break;
      case MotionEvent.ACTION_MOVE:
        int y = (int) (event.getRawY());
        int dy = y - lastY;
        lastY = y;

        if (startNestedScroll(ViewCompat.SCROLL_AXIS_VERTICAL) //如果找到了支持嵌套滚动的父类
            && dispatchNestedPreScroll(0, dy, consumed, offset)) {//父类接受了滑动的请求

          int remain = dy - consumed[1];//获取滚动的剩余距离
          if (remain != 0) scrollBy(0, -remain);

        } else {
          scrollBy(0, -dy);
        }
        break;
    }
    return true;
  }

}

public class NestedScrollingParentLayout extends LinearLayout implements NestedScrollingParent {
.... //省略构造函数
  @Override
  public boolean onStartNestedScroll(View child, View target, int nestedScrollAxes) {
    if (target instanceof NestedScrollingChildView ) return true;
    return false;
  }
  @Override
  public void onNestedPreScroll(View target, int dx, int dy, int[] consumed) {
      scrollBy(0, -dy/2 ); //父控件消耗一半的滑动距离
      consumed[1] = dy/2 ;
  }
}
```
这样便自定义了两个最简单的嵌套滑动控件，这样触摸子控件并进行滑动的时候，父控件也会进行滑动（速度是子控件的一半）。
这种算是最简单的一种，如果想要做成在某一范围内父控件滑动，另一范围内子控件滑动，则需要根据父子控件的 `ScrollY` 来做想要的判断修改。

# Behavior 机制
上面，我们了解了 NestedScrolling 的原理，并做了简单的实现，当然没进行复杂的实现。本节我们先来看看 Behavior 的机制。
```java
public static abstract class Behavior<V extends View> {
    public boolean layoutDependsOn(CoordinatorLayout parent, V child, View dependency) 
    public boolean onDependentViewChanged(CoordinatorLayout parent, V child, View dependency)

    public boolean onStartNestedScroll(View child, View target, int nestedScrollAxes)
    public void onNestedPreScroll(View target, int dx, int dy, int[] consumed)
    public boolean onNestedPreFling(View target, float velocityX, float velocityY)
    public void onNestedScroll(View target, int dxConsumed, int dyConsumed, int dxUnconsumed, int dyUnconsumed)
    public boolean onNestedFling(View target, float velocityX, float velocityY, boolean consumed)
    public void onStopNestedScroll(View target)
    //这里只是列出了几个关键的方法。
}
```
我们知道 behavior 都是配合控件，应用在 CoordinatorLayout 的内部。而 CoordinatorLayout 中的子控件之间可以相互影响，其实就是靠得 Behavior。相互影响分为两种：

    由于某个控件位置的变化，对于别的控件造成的影响。
    由于某个控件滑动（内容位置的变化），对于别的控件造成的影响。
第一种比较好理解，类似于参照物变化了，因此控件本身也发生了变化，以 AppBarLayout.BehaviorScrollingViewBehavior 为例：
```java
public static class ScrollingViewBehavior extends HeaderScrollingViewBehavior {
    ...
    @Override
    public boolean layoutDependsOn(CoordinatorLayout parent, View child, View dependency) {
        // We depend on any AppBarLayouts
        return dependency instanceof AppBarLayout;
    }

    @Override
    public boolean onDependentViewChanged(CoordinatorLayout parent, View child,
            View dependency) {
        offsetChildAsNeeded(parent, child, dependency);
        return false;
    }

    private void offsetChildAsNeeded(CoordinatorLayout parent, View child, View dependency) {
        final CoordinatorLayout.Behavior behavior =
                ((CoordinatorLayout.LayoutParams) dependency.getLayoutParams()).getBehavior();
        if (behavior instanceof Behavior) {
            // Offset the child, pinning it to the bottom the header-dependency, maintaining
            // any vertical gap and overlap
            final Behavior ablBehavior = (Behavior) behavior;
            ViewCompat.offsetTopAndBottom(child, (dependency.getBottom() - child.getTop())
                    + ablBehavior.mOffsetDelta
                    + getVerticalLayoutGap()
                    - getOverlapPixelsForOffset(dependency));
        }
    }
    ...
}
```
首先判断，dependency 是否是是 AppBarLayout，如果是的话，则将其视为依赖对象。每次后者的变化，都会影响到自己，这里便是自己的在y方向上产生偏移。
第二种，则是类似于 NestedScrolling 机制，其实就是使用后者实现的。当子控件要进行滑动时，向父控件询问是否要消耗滑动事件，当滑动完成再次询问父控件，还有什么要做的。流程都是一样的。

## Behavior 实现
```java
public class ToolbarRecyclerBehavior extends CoordinatorLayout.Behavior<Toolbar> {
  private Context mContext;

  public ToolbarRecyclerBehavior() {
  }

  public ToolbarRecyclerBehavior(Context context, AttributeSet attrs) {
    super(context, attrs);
    mContext = context;

    final TypedArray styledAttributes =
        mContext.getTheme().obtainStyledAttributes(new int[] { android.R.attr.actionBarSize });
    mActionBarSize = (int) styledAttributes.getDimension(0, 0);
    styledAttributes.recycle();
  }

  int mActionBarSize;

  float mStart;

  @Override
  public boolean layoutDependsOn(CoordinatorLayout parent, Toolbar child, View dependency) {
    return dependency instanceof RecyclerView;
  }

  @Override
  public boolean onDependentViewChanged(CoordinatorLayout parent, Toolbar child, View dependency) {
    if (mStart == 0) mStart = dependency.getY();

    double perccent = dependency.getY() / mStart;
    int y = (int) (child.getHeight() * (-perccent));
    child.setY(y);
    dependency.setPadding(0, (int) (mActionBarSize * (1 - perccent)), 0, 0);

    return true;
  }
}

// xml 文件
<android.support.design.widget.CoordinatorLayout ...>
  <android.support.design.widget.AppBarLayout
        ...
      >
    <android.support.design.widget.CollapsingToolbarLayout
        ...
        >
      <ImageView
        ...
          />
    </android.support.design.widget.CollapsingToolbarLayout>
  </android.support.design.widget.AppBarLayout>

  <android.support.v7.widget.RecyclerView
      android:id="@+id/recycler"
      app:layout_behavior="@string/appbar_scrolling_view_behavior"
      android:layout_height="match_parent">

  </android.support.v7.widget.RecyclerView>

  <android.support.v7.widget.Toolbar
      ...
      app:layout_behavior="@string/toolbarbehavior"
      />
</android.support.design.widget.CoordinatorLayout>
```
我们定义了一个 `Behavior` ，这个用于 配合 AppBarLayout 和 CollapsingToolbarLayout 对 Toolbar 的隐藏或者显示设置。
# 小结
本篇中，我们从 NestedScrolling 机制入手，了解嵌套滑动的原理，分析了源码中对于嵌套滑动的应用。再到 Behavior 的原理，其实质就是在 NestedScrolling 的基础上进行了扩展。本篇的内容到此结束。



