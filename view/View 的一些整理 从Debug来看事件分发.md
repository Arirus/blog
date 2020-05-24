---
date : 2018-09-19T15:27:56+08:00
tags : ["View", "Android"]
title : "View 的一些整理 从Debug来看事件分发"

---

View 的事件分发又是一个十分经典问题，对于自定义控件来说又是一个绕不开的坎儿。网上关于这方面的文章很多，写的都很细致，但正所谓“纸上得来终觉浅，绝知此事要躬行”，本篇当中，我们结合自定义控件，通过 Debug 的方式来分析下，这个事件分发到底是一个怎样的过程。

### 准备自定义控件 
我们知道到，和事件分发相关的函数主要是有三个：

    // 将事件分发到目标控件，或者这个控件本身就是目标控件，返回 true 就是被本身处理了
    boolean dispatchTouchEvent(MotionEvent event)
    // 用于处理收到的事件，返回 true 就是被本身处理了
    boolean onTouchEvent(MotionEvent event)
    // 用于拦截向外分发的事件，返回 true 就是被本身拦截了
    boolean onInterceptTouchEvent(MotionEvent ev)
这三个方法中，onInterceptTouchEvent(MotionEvent) 是只在 ViewGroup 才会存在的。其三者的关系可以使用伪代码简单的说明：
```java
public boolean dispatchTouchEvent(MotionEvent event){
  boolean consume = false;
  if(onInterceptTouchEvent(event))
    consume = onTouchEvent(event);
  else
    consume = child.dispatchTouchEvent(event);
  
  return consume;
}
```
那我们就来先自定义两个控件，看看其调用顺序：
```java
public class ViewParent extends ViewGroup {
  ...
  @Override
  public boolean dispatchTouchEvent(MotionEvent ev) {
    Log.i(ARIRUS, "dispatchTouchEvent: Parent:"+this);
    return super.dispatchTouchEvent(ev);
  }

  @Override
  public boolean onInterceptTouchEvent(MotionEvent ev) {
    Log.i(ARIRUS, "onInterceptTouchEvent: Parent:"+this);
    return super.onInterceptTouchEvent(ev);
  }

  @Override
  public boolean onTouchEvent(MotionEvent event) {
    Log.i(ARIRUS, "onTouchEvent: Parent:"+this);
    return super.onTouchEvent(event);
  }
}

public class ChildView extends View {
  ...
  @Override
  public boolean dispatchTouchEvent(MotionEvent event) {
    Log.i(ARIRUS, "dispatchTouchEvent: child");
    return super.dispatchTouchEvent(event);
  }

  @Override
  public boolean onTouchEvent(MotionEvent event) {
    Log.i(ARIRUS, "onTouchEvent: child");
    return super.onTouchEvent(event);
  }
}
```
ViewParent 和 ChildView 分别实现了其相应的方法。我们将其布置到布局中，点击有如下日志：
```java
I/ARIRUS: dispatchTouchEvent: Parent:cn.arirus.mddemo.dispatch.ViewParent{29aa5c0 ...... }
I/ARIRUS: onInterceptTouchEvent: Parent:cn.arirus.mddemo.dispatch.ViewParent{29aa5c0 .....}
I/ARIRUS: dispatchTouchEvent: Parent:cn.arirus.mddemo.dispatch.ViewParent{cd942f9 ......}
I/ARIRUS: onInterceptTouchEvent: Parent:cn.arirus.mddemo.dispatch.ViewParent{cd942f9 ......}
I/ARIRUS: dispatchTouchEvent: child
I/ARIRUS: onTouchEvent: child
I/ARIRUS: onTouchEvent: Parent:cn.arirus.mddemo.dispatch.ViewParent{cd942f9 .......}
I/ARIRUS: onTouchEvent: Parent:cn.arirus.mddemo.dispatch.ViewParent{29aa5c0 .......}
```
为了更好的提现 ViewGroup 的事件分发顺序，我放了两个 ViewParent。其调用顺序如上所示。用语言描述就是：ViewParent1 分发事件？-> ViewParent1 不拦截，分发 -> ViewParent2 分发事件？->ViewParent2 不拦截，分发 -> ChildView 收到事件，可否处理？-> ChildView 不能处理，原路返回 -> ViewParent2 不能处理，原路返回 -> ViewParent1 不能处理，原路返回。

我觉得上面的例子就可以很好的展示 View 事件分发体系了，那么后面，我们从 dispatchTouchEvent 入手，看看其究竟是怎样一个过程。

### Debug 分发 ACTION_DOWN 事件
简化起见，我们去除一个 ViewParent，仅保留一个。
#### ViewGroup.dispatchTouchEvent
Debug 开始后，点击 ChildView，ViewGroup 的 dispatchTouchEvent 收到断点。我们可以看到当前 View 是 DecorView 就是根视图了 ![根视图]()，我们可以查看其子控件 ![子控件内容]()，看id知道，一个导航栏，一个状态栏，还有一个 LinearLayout。我们可以猜测那个 LinearLayout 就是主布局了。继续向下：
```java
public boolean dispatchTouchEvent(MotionEvent ev) {
  ...
  final boolean intercepted;
  if (actionMasked == MotionEvent.ACTION_DOWN || mFirstTouchTarget != null) {
      final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
      if (!disallowIntercept) {
          intercepted = onInterceptTouchEvent(ev);
          ev.setAction(action); // restore action in case it was changed
      } else {
          intercepted = false;
      }
  }
}
```
调用了 onInterceptTouchEvent 判断当前是否要拦截。DecorView 决定不拦截，那么继续往下：
```java
if (!canceled && !intercepted) {
    ... //如果是按下操作则继续
    if (actionMasked == MotionEvent.ACTION_DOWN
            || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
            || actionMasked == MotionEvent.ACTION_HOVER_MOVE) 
        ...//如果当前有子控件
        final int childrenCount = mChildrenCount;
        if (newTouchTarget == null && childrenCount != 0) {
            final float x = ev.getX(actionIndex);
            final float y = ev.getY(actionIndex);
            ...//获取点击位置
            final View[] children = mChildren;
            for (int i = childrenCount - 1; i >= 0; i--) { //题外话，这里以倒序的方式来查找很机智啊！
                final int childIndex = getAndVerifyPreorderedIndex(
                        childrenCount, i, customOrder);
                final View child = getAndVerifyPreorderedView(
                        preorderedList, children, childIndex);
                ...// 结合子控件 和 点击位置判断子控件是否具有客观条件来接受 event，即显示，同时点击坐标在控件内部
                if (!canViewReceivePointerEvents(child)
                        || !isTransformedTouchPointInView(x, y, child, null)) {
                    continue;
                }
                // 寻找第一个可触控目标
                newTouchTarget = getTouchTarget(child);
                if (newTouchTarget != null) {
                    // Child is already receiving touch within its bounds.
                    // Give it the new pointer in addition to the ones it is handling.
                    newTouchTarget.pointerIdBits |= idBitsToAssign;
                    break;
                }

                resetCancelNextUpFlag(child);
                // 将 event 分发给 child
                if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                  ...
                }
                ...
        }
```
#### ViewGroup.dispatchTransformedTouchEvent
截止到上面，DecorView 已经大致能确定把不具备接受事件的控件找到并排除出去，现在剩下可以接收事件的子控件，准备对其调用 dispatchTransformedTouchEvent 方法：
```java
private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,View child, int desiredPointerIdBits){
  ...
  if (child == null || child.hasIdentityMatrix()) {
      if (child == null) {
          handled = super.dispatchTouchEvent(event);
      } else {
          final float offsetX = mScrollX - child.mLeft;
          final float offsetY = mScrollY - child.mTop;
          event.offsetLocation(offsetX, offsetY);
          //由于当前的child 是一个 LinearLayout，非空 这样便进入到 dispatchTouchEvent 又会重复上述过程
          handled = child.dispatchTouchEvent(event);

          event.offsetLocation(-offsetX, -offsetY);
      }
      return handled;
  }
  ...
}
```
这样如果，子控件非空，便开始调用子控件的 dispatchTouchEvent ，并执行上述的操作。同样的观察，我们可以知道 LinearLayout 的子控都有谁：![子控件内容]()，同样又在子 FrameLayout 中依次找到了 FitWindowsLinearLayout-> ContentFrameLayout ->ConstraintLayout。这样就到了我们的主布局下面，后面就到了我们的自定义控件里面了。我们直接到 ViewParent.dispatchTouchEvent() 中一探究竟。

#### ViewParent.dispatchTouchEvent
为了和上面区分是不同的ViewGroup，这里使用了 ViewParent 作为标题。dispatchTouchEvent 方法之前的内容都是一样的，直到：
```java
if (actionMasked == MotionEvent.ACTION_DOWN|| mFirstTouchTarget != null) {
    final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
    if (!disallowIntercept) {
        intercepted = onInterceptTouchEvent(ev); // 我们这里实现了 ViewParent.onInterceptTouchEvent 方法
        ev.setAction(action); 
    } else {
        intercepted = false;
    }
}
```
由于我们自己实现了 onInterceptTouchEvent 因此会调用我们的方法。后面又是重复之前的判断，最后进入了 ChildView.dispatchTouchEvent。由于 ChildView 是 View 不是 ViewGroup，因此直接调用了 View.onInterceptTouchEvent 方法。

#### ChildView.onInterceptTouchEvent
```java
public boolean dispatchTouchEvent(MotionEvent event) {
    boolean result = false;
    ...
    ListenerInfo li = mListenerInfo;
    if (li != null && li.mOnTouchListener != null
            && (mViewFlags & ENABLED_MASK) == ENABLED
            && li.mOnTouchListener.onTouch(this, event)) {
        result = true;
    }
    // 判断是否有 TouchListener 如果有则调用之，然后调用 ChildView.onTouchEvent
    if (!result && onTouchEvent(event)) {
        result = true;
    }
    ...
    return result;
}
```
如果都没有处理则返回false给 ViewParent.dispatchTransformedTouchEvent 方法，而后返回到 ViewParent.onInterceptTouchEvent 方法。

#### ViewParent.onInterceptTouchEvent 
```java
public boolean dispatchTouchEvent(MotionEvent ev) {
  ...
  handled = dispatchTransformedTouchEvent(ev, canceled, null,TouchTarget.ALL_POINTER_IDS);
  ...
}
```
由于找不到处理目标，因此传到 null 到 dispatchTransformedTouchEvent 方法中，我们上面看过，如果子控件为空则会直接调用 super.dispatchTouchEvent 方法，而其 super 就是一个 View，因此又调用了 View.dispatchTouchEvent 方法。又会调用了：
```java
public boolean dispatchTouchEvent(MotionEvent event) {
  ...
  ListenerInfo li = mListenerInfo;
  if (li != null && li.mOnTouchListener != null
          && (mViewFlags & ENABLED_MASK) == ENABLED
          && li.mOnTouchListener.onTouch(this, event)) {
      result = true;
  }

  if (!result && onTouchEvent(event)) {
      result = true;
  }
  ...
}
```
其实就是调用了 ViewParent.onTouchEvent 方法。调用结束后返回到 ViewParent.dispatchTouchEvent，然后依次返回上一层等等。这样一个事件就算是完全传递完毕了。

### Debug 分发 ACTION_UP 事件
上节中，我们分析了 ACTION_DOWN 事件的分发过程，为什么 ACTION_DOWN 事件比较特殊？因为 View 处理事件其实是按照事件组来处理的，而一个事件组的开端就是 ACTION_DOWN 事件，如果 ACTION_DOWN 事件都不处理，后面的事件还有怎么处理？就好像“连心爱的女人的...”，手动斜眼。本节中，我们在上面 ACTION_DOWN 没有处理的基础上，来分析 ACTION_UP 会怎样处理。

同 ACTION_DOWN 开始一样，事件一开始也是从 DecorView 传入：
```java
public boolean dispatchTouchEvent(MotionEvent ev) {
  final boolean intercepted;
  if (actionMasked == MotionEvent.ACTION_DOWN || mFirstTouchTarget != null) {
      ...
  } else {
      intercepted = true;
  }
  ...
  if (mFirstTouchTarget == null) {
    handled = dispatchTransformedTouchEvent(ev, canceled, null,TouchTarget.ALL_POINTER_IDS);
  }
}
```
判断当前 mFirstTouchTarget 为空（目前我们还没接触到 mFirstTouchTarget 就先放着）并且不是初始 ACTION_DOWN 事件，直接把当前的事件给拦截下来。
```java
// Dispatch to touch targets.
if (mFirstTouchTarget == null) {
    handled = dispatchTransformedTouchEvent(ev, canceled, null,TouchTarget.ALL_POINTER_IDS);
}    
```
我们知道，调用 dispatchTransformedTouchEvent 并传入 null 子控件，便会调用 View.dispatchTouchEvent 进而调用 View.onTouchEvent 方法， DecorView 直接是会忽略的。因此可以看到当，一个事件没有子控件认领时，便不会传入到子控件内部，这样大大的提升了事件传递效率。

### Debug 消费事件
上面的例子都是在没有拦截消耗事件的基础上进行的，那如果有事件被消费，会是怎样一种情况？我们先将 ChildView.onTouchEvent 返回值修改为 true 来看下。
之前还都是一样的操作从 DecorView.dispatchTouchEvent 开始传入一直到 ViewParent.dispatchTouchEvent 方法。之后在 dispatchTransformedTouchEvent 仿佛在调用了 ChildView.dispatchTouchEvent。上面我们也分析了，ChildView 会返回 ChildView.onTouchEvent 的值，由于我们修改为true，因此会返回告诉父容器已拦截。
```java
public boolean dispatchTouchEvent(MotionEvent ev){
    ...
    if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
        mLastTouchDownTime = ev.getDownTime();
        ...
        mLastTouchDownX = ev.getX();
        mLastTouchDownY = ev.getY();
        newTouchTarget = addTouchTarget(child, idBitsToAssign);
        alreadyDispatchedToNewTouchTarget = true;
        break;
    }
    ...
}

private TouchTarget addTouchTarget(@NonNull View child, int pointerIdBits) {
    final TouchTarget target = TouchTarget.obtain(child, pointerIdBits);
    target.next = mFirstTouchTarget;
    mFirstTouchTarget = target;
    return target;
}
```
如果 dispatchTransformedTouchEvent 返回了 true ，就会把拦截的 child 封装在 mFirstTouchTarget 中，表示有子控件可以接受此事件。这样之后的事件到来，就算其不是 ACTION_DOWN ，但判断 mFirstTouchTarget 非空，也会继续执行 dispatchTransformedTouchEvent 查询子控件谁可以消费这个事件。

上面是我们直接将 ChildView.onTouchEvent 返回为 ture 得到的结果。我们尝试将 ViewParent.onInterceptTouchEvent 返回值改为 ture 来再次尝试。由于拦截但是，ViewParent.onTouchEvent 返回 false，就是拦截未处理，因此之后的事件（ACTION_MOVE, ACTION_UP等）不会再向 ViewParent 中传递。

### 自定义事件分发
我们上面差不多将 View 的事件分发分析的差不多了，那如果我们要自定义控件的事件分发，需要重写哪些函数呢？我们来简单的捋一下。

如果是准备自定义 ViewGroup，三个函数都可以重写，但是对于 dispatchTouchEvent ，建议还是调用 super.dispatchTouchEvent 方法，因为其负责整个事件的分发，如果需要修改的话可以在调用 super 方法之前进行修改。对于 onTouchEvent 与 onInterceptTouchEvent 则关系更为微妙：如果不想子控件来接受事件则重写 onInterceptTouchEvent 返回true，那么此时必须要重写 onTouchEvent 来对事件进行处理，否则仅重写 onTouchEvent 仅当子控件没有处理事件时再进行处理。

对于自定义 View 则相对简单一些，因为没有 onInterceptTouchEvent 方法，类似 ViewGroup 不建议直接修改 dispatchTouchEvent，如果需要拦截事件可以直接修改 onTouchEvent 方法。

### 小结
根据上面的分析，我们可以将 ViewGroup.dispatchTouchEvent 与 View.dispatchTouchEvent 抽象出来，使用如下伪代码来表示。我们就可以清楚的知道，事件在分发的过程是怎么样的。
```java
public boolean ViewGroup.dispatchTouchEvent(MotionEvent ev) {
    if(ACTION_DOWN || FirstTarget !=null){
        intercepted = onInterceptTouchEvent(ev)
    } else
        intercepted = true;

    if(!ACTION_CANCEL && intercepted == false){
        child = foreach childern;
        if (!child.possibleHandle(ev)) continue;

        if(child.dispatchTouchEvent(ev)){
            FirstTarget = child;
            break;
        }
    }

    if(FirstTarget == null)
        handled = View.dispatchTouchEvent(ev);
    else{
        handled = FirstTarget.child.dispatchTouchEvent(ev);
    }

    if(ACTION_CANCEL) resetState();
    return handled;
}

public boolean View.dispatchTouchEvent(MotionEvent ev) {
    result = false;
    if (mOnTouchListener!=null && mOnTouchListener.onTouch(event))
        result = true;
    if (onTouchEvent(ev))
        result = true;
    return result;
}
```
我们可以思考得到一下小结：
    
    事件发布以 DOWN 开始，以 UP 结束；
    onInterceptTouchEvent 表示事件是否可以向后继续传递，而 onTouchEvent 表示事件自己能不能处理，二者是相互独立的；
    当事件被 View 拦截，之后所有的事件 onInterceptTouchEvent 不会再被调用到；
    如果 onTouchEvent 返回 false，之后的事件不会再调用 onTouchEvent；
    自定义 ViewGroup 的 onInterceptTouchEvent 默认不要拦截 DOWN，否则子控件啥也收不到。
    自定义 ViewGroup 注意如果要拦截某个事件时，一定要在 onInterceptTouchEvent 拦截，不能等到子 View 的 onTouchEvent 返回 fasle 后再拦截，这样会收不到拦截。

那么本节的内容就到此为止，下篇再见。