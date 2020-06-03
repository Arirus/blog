---
date : 2018-10-23T9:10:48+08:00
tags : ["View", "Android"]
title : "View 的一些整理 绘制"

---

好了，本篇开始就是 View 绘制相关的内容，因为这里东西比较多且比较杂，因此不免会出现罗列 API 的情况，我们会挑一些重要的 API 来进行说明，对于非重要的 API 有个印象就好。

# Canvas 图形绘制
Canvas（画布）是我们绘制 View 的最重要的两个类之一，我们绘制的图像都是显示在其上面。下面是规范图形常用的 API：
```java
//涂色
//将整个画布涂上颜色，注意如果在 drawColor 之前还有别的绘制，会将其覆盖
Canvas.drawColor(@ColorInt int color) 
//画圆
Canvas.drawCircle(float cx, float cy, float radius, @NonNull Paint paint) 
//画矩形
Canvas.drawRect(float left, float top, float right, float bottom, Paint paint)
Canvas.drawRect(@NonNull Rect r, @NonNull Paint paint) 
Canvas.drawRect(@NonNull RectF r, @NonNull Paint paint) 
//画椭圆
Canvas.drawOval(@NonNull RectF oval, @NonNull Paint paint)
Canvas.drawOval(float left, float top, float right, float bottom, @NonNull Paint paint)
//画直线
Canvas.drawLine(float startX, float startY, float stopX, float stopY, Paint paint)
//画圆角矩形
Canvas.drawRoundRect(float left, float top, float right, float bottom, float rx, float ry, Paint paint) //相比与矩形多了圆角半径
//画圆弧 
//起始角 划过角 是否连到中心连到就是扇形，否则就是圆弧
Canvas.drawArc(@NonNull RectF oval, float startAngle, float sweepAngle, boolean useCenter,@NonNull Paint paint)
Canvas.drawArc(float left, float top, float right, float bottom, float startAngle,float sweepAngle, boolean useCenter, @NonNull Paint paint) 
//画路径
//path 里面也有很多的设置方法，大致 Canvas 能画的图像，path 也都可以画
Canvas.drawPath(@NonNull Path path, @NonNull Paint paint)
```
上面所谓的规范的图像都是圆，线这类的规则图形，有几点可以简单说明下：1.RectF(Rect) 和直接使用 ltrb 性质是一样的，只不过前者把 ltrb 封装成了一个矩形；2.使用 drawOval drawArc 也可以画圆，要根据合适的方法来做图；3.drawXXX 方法只是画图像，至于颜色，填充全都是由 paint 来确定的。

# Paint
上面说了画布，这里就来说画笔 `Paint` ，下面是其常用的 API：
```java
//设置颜色
Paint.setColor(int color) 
//同上
Paint.setARGB(int a, int r, int g, int b) 
//设置着色器
Paint.setShader(Shader shader)
//设置颜色过滤器
Paint.setColorFilter(ColorFilter colorFilter)
//设置转换模式 Xfermode 指的是你要绘制的内容和 Canvas 的目标位置的内容应该怎样结合计算出最终的颜色
Paint.setXfermode(Xfermode xfermode)

Paint.setStyle(Style style) //设置绘制模式
Paint.setStrokeWidth(float width) //设置线条宽度
Paint.setStrokeCap(Paint.Cap cap) // 设置线头形状
Paint.setStrokeJoin(Paint.Join join) // 设置拐角形状

Paint.setDither(boolean dither) //设置图像抖动 常用于图像降低色彩深度绘制时，避免出现大片的色带与色块
Paint.setPathEffect(PathEffect effect) //给图形的轮廓设置效果

Paint.setShadowLayer(float radius, float dx, float dy, int shadowColor) //在之后的绘制内容下面加一层阴影
Paint.setMaskFilter(MaskFilter maskfilter) //是在绘制层上方的附加效果

Paint.setAntiAlias(boolean aa) //设置抗锯齿开关

Paint.getFillPath(Path src, Path dst) //获取实际 Path，在设置了 Style.STROKE 或者 PathEffect 有效 
Paint.getTextPath(String text, int start, int end, float x, float y, Path path)  //获取实际文字路径
Paint.getTextPath(char[] text, int index, int count, float x, float y, Path path)

```
其中 Style 是 enum 类型，有几个可选值：FILL，STROKE，FILL_AND_STROKE;

Shader 着色器总共有 LinearGradient RadialGradient SweepGradient BitmapShader ComposeShader 这几类；TileMode 端点之外的延伸规则：CLAMP,  MIRROR 和 REPEAT；

PorterDuff.Mode 颜色策略总共有17中，需要的时候查文档就好。

ColorFilter 滤镜，总共有三个子类：LightingColorFilter PorterDuffColorFilter 和 ColorMatrixColorFilter

Xfermode 只有一个子类：PorterDuffXfermode ，参数也是 PorterDuff.Mode。使用 Xfermode 时要注意，图像Src与Dst的重叠范围，非重叠范围的Dst图像不会消失。切记图像的重叠范围判断。还有一定要使用离屏缓冲来对图像进行 Xfermode 设置。（为了把需要互相作用的图形放在单独的位置来绘制，不会受 View 本身的影响。如果不使用 saveLayer()，绘制的目标区域将总是整个 View 的范围，两个图形的交叉区域就错误了。） 

PathEffect 路径效果分为两类，单一效果的  CornerPathEffect DiscretePathEffect DashPathEffect PathDashPathEffect ，和组合效果的  SumPathEffect ComposePathEffect。PathDashPathEffect 是用使用另一个 Path 来绘制原 Path 的。

PathDashPathEffect.Style 三个值 TRANSLATE；ROTATE；MORPH。

整个 Paint 与 Canvas 结合绘制图像的 API 差不多就是这些，其中以 PorterDuff.Mode 相关 API 比较复杂，需要好好理解。

# Path 路径绘制
上面我们对 Canvas 绘制规则图像做了总结，这里我们对不规则图像—— Path 来进行下小结。
```java
Path.addXxx() //添加子图形,类似与 Canvas.drawXXX() 方法
Path.addCircle(float x, float y, float radius, Direction dir) // 画圆

Path.xxxTo() //添加一条线

Path.lineTo(float x, float y) 
Path.rLineTo(float x, float y) //画直线,前者是绝对座标，后者是相对座标

Path.quadTo(float x1, float y1, float x2, float y2) 
Path.rQuadTo(float dx1, float dy1, float dx2, float dy2) //画二次贝塞尔曲线

Path.cubicTo(float x1, float y1, float x2, float y2, float x3, float y3)  
Path.rCubicTo(float x1, float y1, float x2, float y2, float x3, float y3) //画三次贝塞尔曲线

Path.moveTo(float x, float y) 
Path.rMoveTo(float x, float y) //移动到目标位置

Path.close() //把当前的子图形封闭,Paint.Style 为 FILL 或  FILL_AND_STROKE Path 会自动封闭起来

Path.setFillType(FillType fillType) //设置填充方式
```
Path.Direction 路径绘制方向：顺时针 (CW clockwise) 和逆时针 (CCW counter-clockwise)

FillType 填充类似总共有4个值：EVEN_ODD WINDING （默认值） INVERSE_EVEN_ODD INVERSE_WINDING；
这个值主要是用来判断图形相交部分的内外性的。

# 文字的测量与绘制
文字和图像不同，主要不同在于两点：文字有一个 baseLine，放置的字符都是以 baseLine 为基础的；绘制字符时，除了字符本身，还会把左右两边带上一些空白，大小和字体字号有关。
## 测量
知道上面这两点，我们就来看看文字测量的 API：
```java
Paint.getFontMetrics(FontMetrics metrics) //FontMetrics 包含了给定的字符串的各种返回值，函数直接返回了建议行间距，使用 breakLine 来分行绘制文字时，可直接使用。
Paint.getTextBounds(String text, int start, int end, Rect bounds) //获取文字的显示范围
Paint.measureText(String text) //测量文字的宽度，这里就是两边会有留白，因此大于上面的 bounds 宽度
Paint.getTextWidths(String text, float[] widths) //获取每个字符的宽度，存入到 widths 中
Paint.breakText(String text, boolean measureForwards, float maxWidth, float[] measuredWidth) //返回给定最大宽度，能绘制到字符索引
```
FontMetrics 里面有 ascent,  descent, top, bottom, leading 5个值。ascent，top 为负值在 baseLine 之上，descent，bottom 为正值在 baseLine 之下。
通常，我们在修正字体在Y轴上的偏移时，可以使用 originY - （ascent + descent）/2 的方式来修正。
当然也可以使用 getTextBounds 返回的 bounds 值来修正，不过后者会随着字符串的变化，偏移值也会变化。

## 绘制
绘制文字的时候都是以 baseLine 为基线进行绘制，由于会出现Y轴向的偏移，因此绘制时要考虑修正偏移。
```java
Paint.drawText(String text, float x, float y, Paint paint) //以 x y 为左侧baseLine起点来绘制 text
Paint.drawTextOnPath(String text, Path path, float hOffset, float vOffset, Paint paint) //以路径为基线进行绘制。hOffset 和 vOffset。它们是文字相对于 Path 的水平偏移量和竖直偏移量，利用它们可以调整文字的位置

Paint.setTextSize(float textSize) //设置文字大小
Paint.setTypeface(Typeface typeface) // 设置字体
Paint.setTextAlign(Paint.Align align) // 设置文字的对齐方式

Paint.setFakeBoldText(boolean fakeBoldText) // 设置粗体
Paint.setStrikeThruText(boolean strikeThruText) //删除线
Paint.setUnderlineText(boolean underlineText) //下划线
Paint.setTextSkewX(float skewX) //斜体
Paint.setTextScaleX(float scaleX) //水平拉宽
```

# Canvas 裁切与变换
```java
Canvas.clipRect(RectF r) // 裁切这个范围的 Canvas，裁切后绘制的图像和之前一样，不过只会显示在这个部分
Canvas.clipPath(Path p) // 同上

Canvas.translate(float dx, float dy) //平移
Canvas.rotate(float degrees, float px, float py) //旋转
Canvas.scale(float sx, float sy, float px, float py) //放缩
Canvas.skew(float sx, float sy) //错切
```
需要说明的是：translate 方法不会改变 canvas 的原点座标，就是说 canvas 移动，原点座标也会跟着移动，例如canvas.translate(-100,-100), 原点也会转换到（-100，-100）的位置，此时绘制 canvas.drawPoint(100,100, paint) 就会在原先的原点处进行绘制。我们再来看看 rotate(float degrees, float px, float py) 方法，其实现为：
```java
public final void rotate(float degrees, float px, float py) {
    if (degrees == 0.0f) return;
    translate(px, py);
    rotate(degrees);
    translate(-px, -py);
}
```
以（x，y）为轴心进行旋转，实现逻辑是先将 canvas 平移（x，y）位置，使得原点位于轴心，旋转后，再将 canvas 平移（-x，-y）回到原来的位置。就像是魔方一样，具体怎么描述我描述不出来，自己体会吧。同时，clipRect 方法给出的 RectF 则是完全不受原点变化的影响，例如之前 clip(100,100,200,200)，如果在平移(-100,-100) 之后，还想显示原先(100,100,200,200)位置的图形，则要剪裁(0,0,100,100)这个位置。

还有一点需要注意，当两个以上 canvas 结合，应该是怎样的一个顺序。rotate 和 scale 都是可以围绕某个轴心来进行变换，如果和 translate 进行结合，先后顺序肯定会影响图像的形状：
```java
    //canvas.translate(EDGE,EDGE);
    canvas.scale(1f,2f);
    canvas.translate(EDGE,EDGE);
    canvas.drawRect(mRectF,mPaint);
```
这里就是，我打算以原点为轴心进行y轴的放大2倍，放大完成后，有个平移（EDGE,EDGE），由于y轴比例放大了，因此y轴向相当于平移了原来 EDGE 的2倍，但是如果我再放大之前进行平移，则不会出现这个情况，x轴 y轴都是征程平移之前的 EDGE 距离。但是如果，我以 mRectF 中心为原点进行旋转，则应该在旋转之后在进行平移，其顺序正好和之前是相反的，因此还是要具体情况具体分析，没有一个通用的解决办法。

# Matrix 变换
Matrix提供了四种操作：translate(平移) rotate(旋转) scale(缩放) skew(错切) ；pre是在队列最前面插入，post是在队列最后面追加，而set先清空队列在添加。这样结合起来有12种方法。但是不建议不同前缀的方法混用，会比较难以理解。canvas.contact(Matrix) 方法来将矩阵和本身结合起来。
```java
matrix.preScale(2f,1f); 
matrix.preTranslate(5f, 0f); 
matrix.postScale(0.2f, 1f); 
matrix.postTranslate(0.5f, 0f); 
```
执行顺序：translate(5, 0) -> scale(2f, 1f) -> scale(0.2f, 1f) -> translate(0.5f, 0f)
如果为了和 canvas 调用 draw 方法的顺序保持一致，可以调用 post 方法，就会依次来进行 canvas 变换。

# Camera 变换
Camera 变换属于三维变换，其模拟在（0，0）处有个 camera，将图像投影到 View 上。需要知道两个方法：
Camera.rotate*() 三维旋转 和 Camera.setLocation(x, y, z) 设置虚拟相机的位置 。最后使用    Camera.applyToCanvas(canvas) 将 camera 和 canvas 相关连起来。
```java
mCamera.save();
mCamera.rotateX(45);
mCamera.applyToCanvas(canvas);
canvas.drawRect(mRectF,mPaint);
mCamera.restore();
```
由于 camera 在左上角因此，看到的图片会是一个非左右对称的图像，因此我们需要把 camera 的焦点对到图像的正中央，因此会这样写：
```java
mCamera.save();
mCamera.rotateX(45);
canvas.translate(getWidth()/2,getHeight()/2); 
mCamera.applyToCanvas(canvas);
canvas.translate(-getWidth()/2,-getHeight()/2); 
canvas.drawRect(mRectF,mPaint);
mCamera.restore();
```
和上面说的canvas变换一样，平移canvas，使得camera移动到图像的中央区域，这时候将camera应用到canvas上，再将canvas移回去。图像就是camera在中央投影下得到的样子。
现在假定我们只想图像显示下半部分，则需要使用 Canvas.clipRect 方法来将下半部分保留下来。想想我们应该在哪里执行裁剪代码：
```java
...
// canvas.clipRect(0,getHeight()/2,getWidth(),getHeight());
canvas.translate(getWidth()/2,getHeight()/2);
mCamera.applyToCanvas(canvas);
canvas.translate(-getWidth()/2,-getHeight()/2);
canvas.clipRect(0,getHeight()/2,getWidth(),getHeight());
canvas.drawRect(mRectF,mPaint);
...
```
如果我们在最上面的注解处进行剪裁，相当于下半部分会出现一个过滤框只会让通过这里的图像显示出来，假如 camera 旋转了超过90度，则出现在下方的应该就是图像的上半部分。如果在 drawRect 之前进行剪裁，则会出现剪裁框随着 camera 的转动而转动，因此在上面绘制图像的下半部分也是会随之转动。这里我就不上图了，读者可以自己在drawRect 后面，填上 `anvas.drawPoint(getWidth()/2,getHeight()/2+EDGE*4,mPaint);` 来验证我们上面的说法。我们也可以把剪裁代码放到 canvas 位移之中，上面说过 clipRect 仅是针对当前的位置进行裁剪，和原点的位置没有任何关系，因此如果要正确显示剪裁内容，clipRect 也要是进行响应的转换：
```java
...
canvas.translate(getWidth()/2,getHeight()/2);
mCamera.applyToCanvas(canvas);
canvas.clipRect(-getWidth()/2,0,getWidth()/2,getHeight()/2);
canvas.translate(-getWidth()/2,-getHeight()/2);
canvas.drawRect(mRectF,mPaint);
...
```
二者的代码是完全一个效果。如果，我们想在这个基础上，进行旋转显示（当前只能显示下半部分，现在我想以图像中心为原点，旋转显示）。这样的话需要旋转 canvas：
```java
...
canvas.translate(getWidth()/2,getHeight()/2);
canvas.rotate(rotDeg);
mCamera.applyToCanvas(canvas);
canvas.clipRect(-getWidth()/2,0,getWidth()/2,getHeight()/2);
canvas.translate(-getWidth()/2,-getHeight()/2);
canvas.drawRect(mRectF,mPaint);
...
```
由于在旋转之前，我们已经做了平移，且在剪切之后又进行了反向平移，因此就相当于在 (getWidth()/2,getHeight()/2) 点处进行旋转。这样连带着 clip 区域和在该区域上绘制的图像都会进行旋转，如果我不想绘制的图像进行旋转则应该在剪裁后绘制前，再将 canvas 进行逆向旋转。
```java
...
canvas.translate(getWidth()/2,getHeight()/2);
canvas.rotate(rotDeg);
mCamera.applyToCanvas(canvas);
canvas.clipRect(-getWidth()/2,0,getWidth()/2,getHeight()/2);
canvas.rotate(-rotDeg);
canvas.translate(-getWidth()/2,-getHeight()/2);
canvas.drawRect(mRectF,mPaint);
...
```
这样得到就是总会只是旋转显示一半的图像。
由于 Camera 的效果比较绕，所以我们用了较大篇幅来说它，使用的时候一定要注意顺序，还有原点位置，切割位置等等。

# 小结
本篇主要是罗列了，绘制时，Canvas，Paint，Path，Camera 等常用实例对象的相关 API，有个大致印象，当我们再次使用时，知道有这么一回事儿，知道往哪个方向查就行。