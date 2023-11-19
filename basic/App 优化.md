# App 优化

## Android 布局优化

耗时原因：
    - xml 解析布局文件（io）方式
    - 反射方式获取类名，并生成相应的控件

方案：
    - include merge viewStub
    - AsynclayoutInflater（google方案），子线程操作，主线程回调
    - 使用ConstraintLayout降低布局嵌套层级
    - 优化过绘
    - 使用 Drawable 渲染，而非 View

    - 线上监控使用LayoutInflaterCompat.setFactory2的方式分别去建立线下全局的布局加载速度和控件加载速度的监控体系，插桩的方式。

## Android 卡顿优化


卡顿原因：
    - 在一个message处理的情况下，处理时间过长，无法在16ms的时间内完成。

方案：
    - Choreographer.postFrameCallback
    - Looper 中的 printer 来进行相应的设置（BlockCanary） 监控卡顿
    - FileObserver可以监听 /data/anr/traces.txt
    - watchDog 通过发送事件来判断阈值时间内是否完成。ANR监控的补充

## Android 内存优化

内存低的原因：Java GC 回收，线程会冻结，影响性能。内存泄漏。内存抖动。OOM

方案
    - 合适的代码设计，数据结构
    - 对象池和缓存池使用
    - 避免内存泄漏 和 内存抖动
    - 资源未关闭，监听器未反注册
    - 内部类持有外部类的引用（Handler相关问题，如持有 Activty 对象）
    - static Activity；View
    - 图片加载优化
      - 选择合适的像素格式
      - 资源文件目录，hdpi越高 占用内存越大
      - BitmapFactory Options 使用，inSampleSize；inJustDecodeBound
      - 大图监控（Glide.onResourceRady）
      - 重复图片检测
      - 线上Bitmap 监控
