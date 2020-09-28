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

