# 基本概念

Drawable是一个抽象的可绘制类。他主要是提供了一个可绘制的区域bound属性以及一个draw成员函数，不同的派生类通过重载draw函数的实现而产生不同的绘制结果。

Drawable类的核心是draw函数的实现，这个函数是一个抽象函数，派生类必须要实现他，函数的入参是一个Canvas画布对象，所有需要绘制的东西都最终绘制到画布上面去：
```java
//抽象的绘制方法
public abstract void draw(Canvas canvas);
```

Drawable在绘制调用draw函数之前必须要先指定绘制的区域，这个区域也是Canvas中要绘制的区域。一旦用户改变了绘制区域时会激发onBoundsChange方法，派生类可以重载onBoundsChange来实现区域变更的处理。
```java
// 获取和设定可绘制区域。
public final Rect getBounds()
public void setBounds(int left, int top, int right, int bottom)
public void setBounds(Rect bounds) 
```

用下面的方法来设置显示的级别，以便进行一些绘制时的区间和条件控制，这个属性并不是所有Drawable派生类都能用到，具体用到的派生类我会在下面说明这个属性的意义。如果设置有变化则会调用onLevelChange，派生类可以重载onLevelChange来实现级别变化的更新处理：
```java
//设置显示的级别，从0到10000
public final boolean setLevel(int level)
public final int getLevel()
```

创建的方式有很多中，这里我们只分析一种：createFromXml，从 xml 中加载。
最后会调用到这个方法。进而调用 DrawableInflater 方法
```java
static Drawable createFromXmlInnerForDensity(@NonNull Resources r,
        @NonNull XmlPullParser parser, @NonNull AttributeSet attrs, int density,
        @Nullable Theme theme) throws XmlPullParserException, IOException {
    return r.getDrawableInflater().inflateFromXmlForDensity(parser.getName(), parser, attrs,
            density, theme);
}

Drawable inflateFromXmlForDensity(@NonNull String name, @NonNull XmlPullParser parser,
        @NonNull AttributeSet attrs, int density, @Nullable Theme theme)
        throws XmlPullParserException, IOException {
    ...//校验判空，暂时不说

    // 直接根据 tag 来进行创建
    Drawable drawable = inflateFromTag(name);
    // tag 中没有的话，可能是用户自定义的，因此尝试使用反射
    if (drawable == null) {
        drawable = inflateFromClass(name);
    }
    drawable.setSrcDensityOverride(density);

    //最后调用 inflate 进行设置数据的读取。基类中 只读取 visible 属性，别的由相应子类各自实现
    drawable.inflate(mRes, parser, attrs, theme);
    return drawable;
}
```
像 setBounds 和 setInstrincWidth 这种方法是没有固定时机的。调用就会设置上的。

# 常见Drawable

## BitmapDrawable
BitmapDrawable 几个属性：
- src：表示该 BitmapDrawable 引用的位图，该图片为 png、jpg 或者 gif；
- antialias：表示是否开启抗锯齿；
- dither：表示当位图和屏幕的像素配置不同时，是否允许抖动。比如一张位图的像素为 ARGB_8888 32 位色，而屏幕像素为 RGB_565；
- filter：是否允许为位图进行滤波以获取平滑的缩放效果；
- gravity：定义位图的 gravity，当位图小于容器时，该属性指定了位图在容器中的停靠位置和绘制方式。
- tileMode：表示当位图小于容器时，执行“平铺”模式，并且指定铺砖的方法。该属性覆盖 gravity 属性——当指定了该属性后，gravity 属性即使设置了，也将不起作用。

BitmapDrawable 放入一个比它大的容器中时，tileMode 就起作用了。
- repeat模式：将重复贴该图，直到填充完容器
- clamp模式：钳位模式，将沿用下边、右边边缘的像素值分水平、垂直两个方向扩展填充剩余位置
- mirror模式：镜像模式，将按水平、垂直镜像重复来填充剩余位置
- disabled：禁用任何填充方法，将使用整个位图进行缩放填充

gravity 则是绘制到不同的位置。

## NinePatchDrawable
这种格式图片的右、下两个边缘的像素点，规定了padding区域，也就是说，内容的绘制时的 padding。

## ClipDrawable
```java
  clip
    |- drawable="@drawable/drawable_id"
    |- clipOrientation="[horizontal | vertical]"
    |- gravity="[ ... ]"
```
根据level来确定绘制的范围。

## LayerDrawable
LayerDrawable可以将一组 Drawable 按 XML 中定义的顺序层叠起来进行绘制，并可以设定每层 Drawable 的 id、位置等等。ProgressBar这个控件的背景切图，可以通过 LayerDrawable 来进行配置。

## AnimationDrawable
```java
animation-list
    |- oneshot="[true | false]"
    |- visible="[true | false]"
    |- item
    |    |- drawable="@drawable/drawable_id"
    |    |- duration="xms"
    |
```
### 原理
AnimationDrawable 只是塞入了一个 View 的 background 中，但是产生动画效果是由于 Drawable.Callback 的回调。

- invalidateDrawable：重绘 Drawable；
- scheduleDrawable：在 when 规定的 ms 后，执行 what 这个Runnable；（这里可以看出动画的端倪了）
- unscheduleDrawable：异步执行这个 what；用来结束动画等。

View 则是实现了 Drawable.Callback 回调。
同时在设置Background时，将相应的回调设置到了上面。start 时，调用setFrame方法：
```java
// drawable
private void setFrame(int frame, boolean unschedule, boolean animate) {
    ...
    selectDrawable(frame);
    if (unschedule || animate) {
        unscheduleSelf(this);
    }
    if (animate) {
        ...
        scheduleSelf(this, SystemClock.uptimeMillis() + mAnimationState.mDurations[frame]);
    }
}

public void scheduleSelf(@NonNull Runnable what, long when) {
    final Callback callback = getCallback();
    if (callback != null) {
        callback.scheduleDrawable(this, what, when);
    }
}

// view
public void scheduleDrawable(@NonNull Drawable who, @NonNull Runnable what, long when) {
    if (verifyDrawable(who) && what != null) {
        final long delay = when - SystemClock.uptimeMillis();
        if (mAttachInfo != null) {
            mAttachInfo.mViewRootImpl.mChoreographer.postCallbackDelayed(
                    Choreographer.CALLBACK_ANIMATION, what, who,
                    Choreographer.subtractFrameDelay(delay));
        } else {
            getRunQueue().postDelayed(what, delay);
        }
    }
}
```
到View的 scheduleDrawable 方法之后，其实就是View.post方法类似了。

这也是帧动画的原理了。


# Tint
其核心在于使用 PorterDuffMode 来混合两个颜色和图形。其底层主要还是用到 ColorFilter。

## Drawable Tint
```java
//Tint 相关设置到这里
@Override
public void setTintList(ColorStateList tint) {
    final BitmapState state = mBitmapState;
    if (state.mTint != tint) {
        state.mTint = tint;
        mBlendModeFilter = updateBlendModeFilter(mBlendModeFilter, tint,
                    mBitmapState.mBlendMode);
        invalidateSelf();
    }
}

@Override
public void setTintBlendMode(@NonNull BlendMode blendMode) {
    final BitmapState state = mBitmapState;
    if (state.mBlendMode != blendMode) {
        state.mBlendMode = blendMode;
        mBlendModeFilter = updateBlendModeFilter(mBlendModeFilter, mBitmapState.mTint,
                blendMode);
        invalidateSelf();
    }
}

//最后会在 View 中进行调用。仅仅会重绘脏区域
@Override
public void invalidateDrawable(@NonNull Drawable drawable) {
    if (verifyDrawable(drawable)) {
        final Rect dirty = drawable.getDirtyBounds();
        final int scrollX = mScrollX;
        final int scrollY = mScrollY;

        invalidate(dirty.left + scrollX, dirty.top + scrollY,
                dirty.right + scrollX, dirty.bottom + scrollY);
        rebuildOutline();
    }
}

// 重绘之后，首先会绘制背景
private void drawBackground(Canvas canvas) {
    final Drawable background = mBackground;
    if (background == null) {
        return;
    }
    ...
    setBackgroundBounds();
    ...
    background.draw(canvas);
}

// 最后会回到 Drawable 中进行绘制
public void draw(Canvas canvas) {
    ...
    if (mBlendModeFilter != null && paint.getColorFilter() == null) {
        paint.setColorFilter(mBlendModeFilter);
        clearColorFilter = true;
    } else {
        clearColorFilter = false;
    }
    ...
}
```

## View Tint
视图中的也类似，以ImageView为例：
```java
private void applyImageTint() {
    if (mDrawable != null && (mHasDrawableTint || mHasDrawableBlendMode)) {
        mDrawable = mDrawable.mutate();

        if (mHasDrawableTint) {
            mDrawable.setTintList(mDrawableTintList);
        }

        if (mHasDrawableBlendMode) {
            mDrawable.setTintBlendMode(mDrawableBlendMode);
        }

        ...
    }
}
```

## mutate
简单讲讲，其实就是如果多个控件使用同一个Drawable，如果其中一个控件的Drawable发生改变，其他所有的Drawable都会发生改变。如果使用Drawable.mutate()，就可以从Drawable里新建一个不可变的实例，那么当这个Drawable发生改变时，不会导致其他的Drawable发生改变。


