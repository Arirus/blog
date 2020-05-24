---
date : 2018-10-29T16:06:53+08:00
tags : ["View", "Android","属性动画"]
title : "View 的一些整理 属性动画"

---

前面的几篇，我们差不多把 View 测量，布局，绘制的各个步骤都分析了一边，本篇我们就在绘制的基础上来看看属性动画的使用。本篇不会涉及帧动画和补间动画，这两种单独放到一篇中来进行说明。

### ViewPropertyAnimator
这个类是一个自动组织和优化一个 View 中的某几个属性的辅助类。使用很简单例如：
```java
mTextView.animate().translationX(Utils.dp2px(100)).start();
```
View.animate() 获取 ViewPropertyAnimator 对象，可以针对 TranslationX/Y/Z、X/Y/Z、rotation/X/Y、ScaleX/Y、Alpha 设置 to 方法和 by 方法。这几个属性是 View 中的，因此可以完全无视是否是自定义的视图，总是可以实现的。但是也有几点是无法实现的

    不同属性只能同时变化，无法做到先后执行；
    只有上面几个属性有动画支持，自定义属性无法支持。

针对这两个缺点，我们可以使用 ObjectAnimator 来进行属性动画。

### ObjectAnimator
ObjectAnimator 的使用方式很简单：

    如果是自定义控件，需要添加 setter / getter 方法；
    用 ObjectAnimator.ofXXX() 创建 ObjectAnimator 对象；
    用 start() 方法执行动画。
我们这里给出一个简单的例子：
```java
ObjectAnimator objectAnimator = ObjectAnimator.ofFloat(mTextView, "translationX" ,100);
objectAnimator.start();
```
这里便是一个修改 translationX 的动画版本。这上面我们就能大致明白 ObjectAnimator 这个原理：`针对 mTextView 的 translationX 属性。来进行变化修改最终达到100`，类似的方法有：
```java
ObjectAnimator.ofObject(Object target, String propertyName, TypeEvaluator evaluator, Object... values) 
ObjectAnimator.ofArgb(Object target, String propertyName, int... values) 
ObjectAnimator.ofInt(Object target, String propertyName, int... values)
ObjectAnimator.ofFloat(Object target, String propertyName, float... values) 
ObjectAnimator.ofPropertyValuesHolder(Object target, PropertyValuesHolder... values)
```
这里只给出了每个类型方法的基本版本，其基本原理就是:`针对 target 的 propertyName 属性，做数值变换最终达到 values`。

除了上面的生成方法，还有一些 ObjectAnimator 的设置方法：
```java
ObjectAnimator.setDuration(int duration) //设置动画时长
ObjectAnimator.setInterpolator(Interpolator interpolator) //设置插值器
ObjectAnimator.addListener(Animator.AnimatorListener listener) // 设置监听器
```
Interpolator 是一个速度模型，是事件和速度的对应关系。基本的插值器有：AccelerateDecelerateInterpolator AccelerateInterpolator DecelerateInterpolator LinearInterpolator 等等。

#### TypeEvaluator 
类型计算器，我们上面举的例子都是以 int float 为例，SDK 会自动计算其中间值。但是如果我们的属性参数类型为非基本类型，SDK 是无法为我们计算出中间值的，此时就要使用 TypeEvaluator 由我们手动计算出中间值。这里我们给出一个例子：
```java
public class AnimCircleView extends View {
  PointF ANCHOR_POINT;
  PointF mDstPoint;
  Paint mPaint;
  final static float EDGE =Utils.dp2px(100);

  {
    ARC_POINT = new PointF(EDGE,EDGE);
    mPaint = new Paint(Paint.ANTI_ALIAS_FLAG);
    mPaint.setStyle(Paint.Style.STROKE);
    mPaint.setStrokeWidth(Utils.dp2px(2));
    mDstPoint = new PointF(EDGE*2,EDGE*2);
  }

  public AnimCircleView(Context context, @Nullable AttributeSet attrs) {
    super(context, attrs);
  }

  public PointF getDstPoint() {
    return mDstPoint;
  }

  public void setDstPoint(PointF dstPoint) {
    mDstPoint = dstPoint;
    invalidate();
  }

  @Override
  protected void onDraw(Canvas canvas) {
    float rad = (mDstPoint.length()-ANCHOR_POINT.length())/2;

    float x = (ANCHOR_POINT.x+mDstPoint.x)/2;
    float y = (ANCHOR_POINT.y+mDstPoint.y)/2;

    canvas.drawCircle(x,y,rad,mPaint);
    canvas.drawCircle(ANCHOR_POINT.x,ANCHOR_POINT.y,Utils.dp2px(2),mPaint);
    canvas.drawCircle(mDstPoint.x,mDstPoint.y,Utils.dp2px(2),mPaint);
  }
}
```
自定义一个 View ，给定一个锚点，另一个是运动的，两个点之间就是园所在的圆心位置，如果我们想修改另一个点的位置达到动态变化，可以进行如下调用：
```java
ObjectAnimator objectAnimator =
    ObjectAnimator.ofObject(mAnimCircleView, "DstPoint", new PointFEvalutor(),
        new PointF(Utils.dp2px(300), Utils.dp2px(300)));
objectAnimator.setDuration(5000).setStartDelay(3000);
objectAnimator.start();

private static class PointFEvalutor implements TypeEvaluator<PointF> {
    PointF mPointF = new PointF();
    @Override
    public PointF evaluate(float fraction, PointF startValue, PointF endValue) {
      mPointF.x = startValue.x + (endValue.x - startValue.x) * fraction;
      mPointF.y = startValue.y + (endValue.y - startValue.y) * fraction;
      return mPointF;
    }
}
```
这里我们自定义了一个 PointFEvalutor 实现了 TypeEvaluator 方法，则每次目的点发生变化时，都会掉要 evaluate 方法，我们可以根据 fraction 来计算出需要变化的值，这样便获取到了进行中点的位置。效果如下：

![TypeEvaluator 使用](http://opf280xl2.bkt.clouddn.com/view_anim.gif?imageMogr2/thumbnail/!30p/blur/1x0/quality/75|watermark/2/text/5ZSv5LiA5oyH5a6a5Y2a5a6iOiBhcmlydXMuY24=/font/5a6L5L2T/fontsize/240/fill/IzA2MDY5OA==/dissolve/100/gravity/SouthWest/dx/10/dy/10|imageslim)

#### PropertyValuesHolder
上面我们使用 ObjectAnimator，都是一个 ObjectAnimator 修改一个属性，使用 PropertyValuesHolder 可以同时对于一个 ObjectAnimator 改变多个属性，类似于使用 ViewPropertyAnimator 多个属性同时变化：
```java
PropertyValuesHolder valuesHolder =
    PropertyValuesHolder.ofObject("DstPoint", new PointFEvalutor(),
        new PointF(Utils.dp2px(300), Utils.dp2px(300)));
PropertyValuesHolder valuesHolder1 =
    PropertyValuesHolder.ofFloat("alpha",0.1f);

ObjectAnimator objectAnimator = ObjectAnimator.ofPropertyValuesHolder(mAnimCircleView,valuesHolder,valuesHolder1);
objectAnimator.setDuration(3000).setStartDelay(3000);
objectAnimator.start();
```
ObjectAnimator 此时可以在修改 DstPoint 属性的同时修改 alpha 属性。
##### Keyframe
Keyframe 关键帧，可以被用来定义`当动画完成度达到某处时，所对应的参数完成的值`。先看其用法：
```java
PropertyValuesHolder alphaValuesHolder =
    PropertyValuesHolder.ofKeyframe("alpha",
        Keyframe.ofFloat(0,1f),
        Keyframe.ofFloat(0.8f,0.9f),
        Keyframe.ofFloat(1f,0.1f)
    );
```
这里我们给出了三个关键帧：动画完成度为0时，alpha值为1；动画完成度为80%时，alpha值为0.9；动画完成度为100%时，alpha值为1。就是说，前80%的过程，alpha几乎没啥变化，最后20%，alpha变化比较剧烈。注意`定义关键帧和设置插值器完全不冲突`，这么说是因为插值器给出了时间完成度和动画完成度的相互关系，跟实际上属性的变化没有关系，属性变化的中间值被设置到了关键帧里面了。
### AnimatorSet
上面我们说了，ViewPropertyAnimator 不能做到不同属性依次变化产生的动画效果，同样由于 ObjectAnimator 由于只是一个动画对象，在不借助外部的情况下也无法做成多个 ObjectAnimator 依次生效。AnimatorSet 就是为了解决这种情况而产生的，可以同时或者依次使得不同 ObjectAnimator 来执行。
```java
AnimatorSet animatorSet = new AnimatorSet();
animatorSet.playTogether(animator1, animator2);
animatorSet.start();
```
这样就可以同时执行 animator1, animator2 了，类似的还有 playSequentially 方法是依次执行，这里我们就不将其所有 API 都列出了。

### 小结
本节主要是通过两个类：ViewPropertyAnimator 和 ObjectAnimator 来研究了一下属性动画，其本质就是在一个变化的过程中，每时每刻都在改变某个属性值并且重绘，从而达到一种“动态变化的效果”。针对属性为非基本类型的控件，我们可以通过调用 ofObject 并且实现 TypeEvaluator 类，来获得相应的 ObjectAnimator。当一个 ObjectAnimator 有多个属性要同时变化可以使用 PropertyValuesHolder 对属性进行包装。最后当多个 ObjectAnimator 要同时或者依次执行时，使用 AnimatorSet 来统一管理。
