# 动画

# 补间动画
 
## 创建
```java
// 通常都是使用 AnimationUtils
//也是从 xml 文件里来读取的

private static Animation createAnimationFromXml(Context c, XmlPullParser parser,
            AnimationSet parent, AttributeSet attrs) throws XmlPullParserException, IOException {
    
    ...

    while (((type=parser.next()) != XmlPullParser.END_TAG || parser.getDepth() > depth)
            && type != XmlPullParser.END_DOCUMENT) {

        if (type != XmlPullParser.START_TAG) {
            continue;
        }

        String  name = parser.getName();

        if (name.equals("set")) {
            anim = new AnimationSet(c, attrs);
            createAnimationFromXml(c, parser, (AnimationSet)anim, attrs);
        } else if (name.equals("alpha")) {
            anim = new AlphaAnimation(c, attrs);
        } else if (name.equals("scale")) {
            anim = new ScaleAnimation(c, attrs);
        }  else if (name.equals("rotate")) {
            anim = new RotateAnimation(c, attrs);
        }  else if (name.equals("translate")) {
            anim = new TranslateAnimation(c, attrs);
        } else if (name.equals("cliprect")) {
            anim = new ClipRectAnimation(c, attrs);
        } else {
            throw new RuntimeException("Unknown animation name: " + parser.getName());
        }

        if (parent != null) {
            parent.addAnimation(anim);
        }
    }

    return anim;

}
```
主要就是这么几类，因此对于补间动画其实是不支持自定义的。

## 启动
```java
// 启动调用 animation，设置并重绘
public void startAnimation(Animation animation) {
    animation.setStartTime(Animation.START_ON_FIRST_FRAME);
    setAnimation(animation);
    invalidateParentCaches();
    invalidate(true);
}

// 重绘后到 view 方法
boolean draw(Canvas canvas, ViewGroup parent, long drawingTime) {
    applyLegacyAnimation(parent, drawingTime, a, scalingRequired);
}

public boolean getTransformation(long currentTime, Transformation outTransformation) {
    if (mStartTime == -1) {
        mStartTime = currentTime;
    }

    final long startOffset = getStartOffset();
    final long duration = mDuration;
    float normalizedTime;
    // 计算流逝百分比
    if (duration != 0) {
        normalizedTime = ((float) (currentTime - (mStartTime + startOffset))) /
                (float) duration;
    } else {
        // time is a step-change with a zero duration
        normalizedTime = currentTime < mStartTime ? 0.0f : 1.0f;
    }

    final boolean expired = normalizedTime >= 1.0f || isCanceled();
    mMore = !expired;

    if (!mFillEnabled) normalizedTime = Math.max(Math.min(normalizedTime, 1.0f), 0.0f);

    if ((normalizedTime >= 0.0f || mFillBefore) && (normalizedTime <= 1.0f || mFillAfter)) {
        if (!mStarted) {
            fireAnimationStart();
            mStarted = true;
            if (NoImagePreloadHolder.USE_CLOSEGUARD) {
                guard.open("cancel or detach or getTransformation");
            }
        }

        if (mFillEnabled) normalizedTime = Math.max(Math.min(normalizedTime, 1.0f), 0.0f);

        if (mCycleFlip) {
            normalizedTime = 1.0f - normalizedTime;
        }
        // 计算插值器的值
        final float interpolatedTime = mInterpolator.getInterpolation(normalizedTime);
        applyTransformation(interpolatedTime, outTransformation);
    }


    return mMore;
}

// 相应的 anmation子类的实现，非基类实现。
protected void applyTransformation(float interpolatedTime, Transformation t) {
    final float alpha = mFromAlpha;
    t.setAlpha(alpha + ((mToAlpha - alpha) * interpolatedTime));
}
```

## 动画优化建议

- 使用最合理的实现方式
    考虑动画的性能：窗口动画>属性动画（View.animate 和 ActivityOptions 转厂动画）>补间动画>draw动画（draw/scroll invalidate），帧动画>Layout动画
- 排查不必要的UI线程耗时
    配合 Android Profiler的CPU火焰图，都能较快的定位到耗时的函数。
- 关注首帧和尾帧绘制
    AnimatorListener的回调都是发生的doFrame，AnimatorListener的回调都是发生的doFrame对于onAnimationEnd来说，回调触发时，最后一帧还没有绘制，避免耗时操作出现。
    对于首帧。保证界面先draw一次，再开始动画。

