---
date : 2018-09-27T17:31:33+08:00
tags : ["View", "Android"]
title : "View 的一些整理 从Debug来看工作流程"

---

从本篇开始，我们开始进入 View 的底层研究，本节我们先来最基本的工作原理入手，了解 View 测量，布局，绘制的一系列流程。

# 梦开始的地方

我们知道 Android 中，视图是以视图树的形式将根视图和子视图连接在了一起，在《事件分发篇》中，我们说过 Android 的根视图一个叫 DecorView 的 ViewGroup。而 DecorView 的一些列操作是由谁来调用的呢？其实它是由一个叫 ViewRootImpl 的对象来调用的。注意 ViewRootImpl 本身既非 View 也非 ViewGroup ，不过他实现了 ViewParent 接口，同时官方在将其描述为 `The top of a view hierarchy` ，因此根视图中的事件分发都是由 ViewRootImpl 来完成的。

当收到的通知，performTraversals(), 便开始变量视图树执行测量，布局，绘制的操作：

```java
private void performTraversals() {
  ...
  measureHierarchy(host, lp, res, desiredWindowWidth, desiredWindowHeight);
  ...
  performLayout(lp, mWidth, mHeight);
  ...
  performDraw();
  ...
}
```

方法中一次通过这三个函数来对视图树的 View 进行测量，布局，绘制操作。
我们接下来通过几个例子来分析下，三个函数的使用过程。

# 测量

```java
<FrameLayout
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    >

  <TextView
      android:layout_width="200dp"
      android:layout_height="200dp"
      android:layout_gravity="left|center"
      />

  <TextView
      android:layout_width="wrap_content"
      android:layout_height="wrap_content"
      android:layout_gravity="right|center"
      />

</FrameLayout>
```

我们定义了一个 FrameLayout 布局，里面放置两个 TextView，Debug 后，会运行到，FrameLayout.onMeasure 方法中。那么测量就从这里开始。

## MeasureSpec

在正式开始读 onMeasure 之前，我们先看下 MeasureSpec 类，MeasureSpec封装了从父级传递给子级的布局要求，也就是控件的 layout_width，layout_height 属性都会封装在这个类里面。其本质是一个32位长度的二进制的int型，最高两位表示 mode，后面三十位表示size。
其中 mode 分为：

    UNSPECIFIED // 父控件没有对子控件施加任何限制，子控件可以想变成多大都可以；
    EXACTLY // 父控件已经对子控件的大小决定好了，子控件按照这个类型执行；
    AT_MOST // 子控件最大可以达到给定的大小。
所以，在 onMeasure 函数中接收到两个参数：widthMeasureSpec, heightMeasureSpec 就可以根据父控件的要求结合自身 LayoutParams，来确定最后的大小。当 layout_width，layout_height 为 MATCH_PARENT，或者给出了具体的长度时，mode 会是 EXACTLY，当 layout_width，layout_height 为 WRAP_CONTENT，mode 则是 AT_MOST，而对于 UNSPECIFIED 一般用不到，只是系统内部使用，不用考虑。

## View.onMeasure

onMeasure 方法中就是进行测量，并最终决定宽高的函数。当测量完成后一定要调用 setMeasuredDimension 函数来将测量的宽高储存起来。

```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(),widthMeasureSpec),
            getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
}
```

上面是 View 中 onMeasure 的实现，函数很简单从字面意思就是直接存储宽高的默认的 size，我们可以看下 getDefaultSize 函数：

```java
public static int getDefaultSize(int size, int measureSpec) {
    int result = size;
    int specMode = MeasureSpec.getMode(measureSpec);
    int specSize = MeasureSpec.getSize(measureSpec);

    switch (specMode) {
    case MeasureSpec.UNSPECIFIED:
        result = size;
        break;
    case MeasureSpec.AT_MOST:
    case MeasureSpec.EXACTLY:
        result = specSize;
        break;
    }
    return result;
}
```

当 mode 为 EXACTLY 或者 AT_MOST，defaultSize 为测量的 size，因此假如自定义了一个 View 同时没有重写 onMeasure 方法，则使用 WRAP_CONTENT 和 MATCH_PARENT 是一样的效果。最后使用了 setMeasuredDimension 将测量后的宽高储存了起来。

因此我们可以这样理解 View.onMeasure 方法：直接将 MeasureSpec 的 size 部分作为最终值储存了起来，同时也忽略了 WRAP_CONTENT 直接使用 MATCH_PARENT 的效果。同时由于 ViewGroup 没有重写 measure/onMeasure 方法，因此当自定义 ViewGroup 如果没有重写 onMeasure 函数，便会执行 View.onMeasure 只会测量自己本身重复上述的效果，并且不会测量子控件。当 setMeasuredDimension 调用后，控件的宽高便会存到 mMeasuredWidth 和 mMeasuredHeight ，这样一个控件的测量便完成了。

细心的我们会想到在上一篇《嵌套滑动的解决方案》中，我们自定义了两类控件一个父控件可以左右滑动，一个子控件可以上下滑动，但是我们并没有重写任何的 onMeasure 方法，按照这里的理论就是父控件只测量了自己并没有测量子控件，那么子控件的 mMeasuredWidth 和 mMeasuredHeight 都应该是为 0 ，但为什么最后子控件还有宽高呢？这里我们先挖个坑，下面再说。

## FrameLayout.onMeasure

上面我们已经理解了 View.onMeasure 方法的原理和思路，这里我们就结合 FrameLayout 来看看一个 ViewGroup 如果要重写 onMeasure 是怎样的一个思路。

```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
  // 从字面的意思很好理解：是否测量 MatchParent 的控件。
  // 这个特性是针对多个子控件使用了 match_parent 来对子控件进行重新测量，至于为啥要这么做，我不太理解。。
  final boolean measureMatchParentChildren =
                MeasureSpec.getMode(widthMeasureSpec) != MeasureSpec.EXACTLY ||
                MeasureSpec.getMode(heightMeasureSpec) != MeasureSpec.EXACTLY;
  ...
  //这里便是遍历孩子对孩子进行测量
  for (int i = 0; i < count; i++) {
      final View child = getChildAt(i);
      if (mMeasureAllChildren || child.getVisibility() != GONE) {
          measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0);
          final LayoutParams lp = (LayoutParams) child.getLayoutParams();
          maxWidth = Math.max(maxWidth,
                  child.getMeasuredWidth() + lp.leftMargin + lp.rightMargin);
          maxHeight = Math.max(maxHeight,
                  child.getMeasuredHeight() + lp.topMargin + lp.bottomMargin);
          childState = combineMeasuredStates(childState, child.getMeasuredState());
          if (measureMatchParentChildren) {
              if (lp.width == LayoutParams.MATCH_PARENT ||
                      lp.height == LayoutParams.MATCH_PARENT) {
                  mMatchParentChildren.add(child);
              }
          }
      }
  }
}
```

这里我们先不关注 measureChildWithMargins 函数，先看整个流程。对每个子控件调用 measureChildWithMargins 来测量每个孩子的宽高。最后选取当下子控件和最大size下的最大值作为新的最大值，依次进行直到所有子控件都被测量，测量完孩子后判断这个子控件是否在 FrameLayout 为 wrap_content 的条件下是 match_parent ，这样会出现我们上面说的逻辑悖论，这样先把这个孩子储存起来，等最后处理。

```java
...
setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
        resolveSizeAndState(maxHeight, heightMeasureSpec,childState << MEASURED_HEIGHT_STATE_SHIFT));
...
```

在对所有子控件进行测量后，便进行了 setMeasuredDimension 。我们来看看 resolveSizeAndState 函数：

```java
public static int resolveSizeAndState(int size, int measureSpec, int childMeasuredState) {
    final int specMode = MeasureSpec.getMode(measureSpec);
    final int specSize = MeasureSpec.getSize(measureSpec);
    final int result;
    switch (specMode) {
        case MeasureSpec.AT_MOST:
            if (specSize < size) {
                result = specSize | MEASURED_STATE_TOO_SMALL;
            } else {
                result = size;
            }
            break;
        case MeasureSpec.EXACTLY:
            result = specSize;
            break;
        case MeasureSpec.UNSPECIFIED:
        default:
            result = size;
    }
    return result | (childMeasuredState & MEASURED_STATE_MASK);
}
```

这函数主要是通过 MeasureSpec 约束条件来协调所需要的大小和状态。这个其实子控件所需要的大小和父控件能提供的大小综合确定的一个函数。其基本意思是：当父控件是 EXACTLY 时，那就按照 MeasureSpec 的 size 来，测量的孩子大小完全忽略；当父控件是 AT_MOST 时，取子控件所需大小和父控件能提供大小的较小值。最后将比较的结果储存起来，那么父控件的大小就确定了下来。整个思路就是先确定所有的子控件的大小，最后根据父控件 mode 来确定父控件的大小。那么正常的测量逻辑就到这里结束了，一句话来说就是通过测量子控件的宽高来决定父控件的宽高。

## measureChildWithMargins

我们上面暂时掠过了 measureChildWithMargins 函数，这里我们来看看其究竟是怎么测量的子控件。

```java
protected void measureChildWithMargins(View child, int parentWidthMeasureSpec, int widthUsed,
        int parentHeightMeasureSpec, int heightUsed) {
    final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
            mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                    + widthUsed, lp.width);
    final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
            mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                    + heightUsed, lp.height);

    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
```

方法很好理解，根据给定的 parentMeasureSpec 来确定 childMeasureSpec，最后让子控件自己进行测量。我们来看看 getChildMeasureSpec 函数：

```java
    public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
        int specMode = MeasureSpec.getMode(spec);
        int specSize = MeasureSpec.getSize(spec);

        int size = Math.max(0, specSize - padding);

        int resultSize = 0;
        int resultMode = 0;

        switch (specMode) {
        case MeasureSpec.EXACTLY:
            if (childDimension >= 0) {
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // 子视图想和父视图一样大
                resultSize = size;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // 子视图要自己决定大小，但是其大小不能超过父视图
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        case MeasureSpec.AT_MOST:
            if (childDimension >= 0) {
                // 子视图有规定的大小，就他了！
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // 子视图想和父视图大小一样，但是父视图没定，所以子视图不能大于父视图
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // 子视图想要自己决定大小，但是不能比父视图大。
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;
            ...
        }
        //noinspection ResourceType
        return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
    }
```

方法先将 parentMeasureSpec 获取 mode 与 size，最后根据父视图的 mode 和子视图的需求来确定子视图的mode 和 size 最后，获取 childMeasureSpec 就可以了。上面我们可以总结两条：

    1. 子视图 childDimension == LayoutParams.MATCH_PARENT 时，会根据父视图的 mode 来确定子视图的 mode。其余都是直接根据 childDimension 来决定子视图的 mode。
    2. 当子视图的 mode 为 AT_MOST，其 size 就是 parentMeasureSpec 的 size。

那么测量就是这样的一个流程：**对于 View 测量时，默认将 MeasureSpec 的 size 作为最后的测量值，mode 是 match_parent，ViewGroup 则需要通过测量每一个子控件来确定其本身的大小**。

# 布局

上面测量完成后，就开始布局了，我们直接从 View.layout 方法开始。

## View.layout

```java
public void layout(int l, int t, int r, int b) {
    ...
    int oldL = mLeft;
    int oldT = mTop;
    int oldB = mBottom;
    int oldR = mRight;

    boolean changed = isLayoutModeOptical(mParent) ?
            setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

    if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
        onLayout(changed, l, t, r, b);
    }
}
```

这里我们只保留关键代码，看到其实控件其实是调用了 setFrame 方法，来确定 mLeft, mTop, mBottom, mRight 4个值。最后将 l, t, r, b 四个值传入 onLayout ，那么控件就可以进行响应的修改来更改布局参数。因为当 mLeft, mTop, mBottom, mRight 四个值都确定了，视图的位置也就确定，onLayout 在 View 中也就没有必要实现，如果自定义的控件需要修改其布局则可以重写 onLayout 方法。由于类似于 ViewGroup.onMeasure 方法，需要先通过测量子控件的宽高进而确定 ViewGroup 的宽高，ViewGroup.onLayout 里面也要手动对子控件进行布局，那这里让我们看下 FrameLayout 是怎么实现的 onLayout 方法。

## FrameLayout.layoutChildren

onLayout 方法中直接调用了 layoutChildren 方法。我们直接来看：

```java
void layoutChildren(int left, int top, int right, int bottom, boolean forceLeftGravity) {
    final int count = getChildCount();
    //获取父控件的 ltrb 那么子控件应该在这个范围内。
    final int parentLeft = getPaddingLeftWithForeground();
    final int parentRight = right - left - getPaddingRightWithForeground();

    final int parentTop = getPaddingTopWithForeground();
    final int parentBottom = bottom - top - getPaddingBottomWithForeground();

    for (int i = 0; i < count; i++) {
        final View child = getChildAt(i);
        if (child.getVisibility() != GONE) {
            final LayoutParams lp = (LayoutParams) child.getLayoutParams();

            // 获取zi控件测量宽高 用于确定占地位置
            final int width = child.getMeasuredWidth();
            final int height = child.getMeasuredHeight();

            int childLeft;
            int childTop;
            ...
            // 判断水平向的 gravity
            switch (absoluteGravity & Gravity.HORIZONTAL_GRAVITY_MASK) {
                case Gravity.CENTER_HORIZONTAL:
                    childLeft = parentLeft + (parentRight - parentLeft - width) / 2 +
                    lp.leftMargin - lp.rightMargin;
                    break;
                case Gravity.RIGHT:
                    if (!forceLeftGravity) {
                        childLeft = parentRight - width - lp.rightMargin;
                        break;
                    }
                case Gravity.LEFT:
                default:
                    childLeft = parentLeft + lp.leftMargin;
            }
            // 判断垂直向的 gravity
            switch (verticalGravity) {
                case Gravity.TOP:
                    childTop = parentTop + lp.topMargin;
                    break;
                case Gravity.CENTER_VERTICAL:
                    childTop = parentTop + (parentBottom - parentTop - height) / 2 +
                    lp.topMargin - lp.bottomMargin;
                    break;
                case Gravity.BOTTOM:
                    childTop = parentBottom - height - lp.bottomMargin;
                    break;
                default:
                    childTop = parentTop + lp.topMargin;
            }
            //之后调用子视图让其进行布局就可以了。
            child.layout(childLeft, childTop, childLeft + width, childTop + height);
        }
    }
}
```

我们知道 FrameLayout 默认将所有的子视图都放置到左上角，通过这里可以明显看出，不设置 gravity 时就和设置了 `left|top` 性质是一样的，默认的将所有子控件全都放置到左上角。最后我们知道 layout(l, t, r, b) 就是在 setFrame 方法中，将 ltbr 和之前的 mLeft, mTop, mBottom, mRight 进行比较并进行赋值。

好了，我们可以解决上面提得问题为什么在一个子控件中没有重写 onMearsure 方法，但是还是可以绘制出其样式？原因在于父控件实现了 onLayout 方法，将响应的 ltrb 传入了 child.layout 方法（注意不是 onLayout，那个是 protected ），最后导致了 mLeft, mTop, mBottom, mRight 被赋值，那么子控件就无需测量的宽高而有了位置点和宽高值，因为 mWidth 和 mHeight 都是通过计算出来的。

因此我们也常说：**onMeasure 之后可以获取测量宽高，onLayout 之时我们可以获取实际宽高，测量宽高仅供参考，一切请以实际宽高为准**。真的不要太现实。

# 绘制

到最后一步了，绘制。View 中，绘制的方法是 draw(canvas) ，那我们先来看看 draw 方法的一个流程：

```java
public void draw(Canvas canvas) {
    ...
    // 当不需要绘制边界褪色的情况下按照如下方式进行绘制
    // 绘制背景
    drawBackground(canvas);
    ...
    // 绘制内容, 绘制子视图
    onDraw(canvas);
    dispatchDraw(canvas);
    ...
    // 绘制qianjing
    onDrawForeground(canvas);
}
```

这里说的边界褪色（Edge fading）是一种特殊效果，类似于：![edge fading](view_edge_fading.png),这里我们暂且不考虑这种绘制。

## View.onDraw

在这个方法中，我们可以实现响应绘制，画方，画圆都可以，最终都是通过画布（canvas）来实现的，由于这里方法过多，我们另开一篇来写，这里仅给出一个例子：

```java
@Override
protected void onDraw(Canvas canvas) {
int width = getWidth();
int height = getHeight();
canvas.drawCircle(width/2,height/2,width/2,new Paint());
}
```

这个就是在控件正中间的位置，以宽度一半为半径来进行一个画圆的操作。

# 小结

本篇中，我们着重从源码出发，了解 View 的工作流程包括视图的测量、布局、绘制。整体来说就是贯彻一个思路：**测量、布局、绘制依次进行，从上到下布置任务，从下到上给出反馈**。有几点需要注意：自定义控件要重写 onMeaseure 方法，父控件还要重写 onLayout，onDraw 需要绘制便要重写。好了，那本篇的内容先到此结束，下篇见。