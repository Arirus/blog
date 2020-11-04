# 帧数记录

- 利用 FrameCallback 的 doFrame 回调。
- 使用 Message.Printer。通过对 MessageQueue 中每一个 Message 的前后进行记录，打到监控性能的目的。（BlockCanry）

# MainThread 和 RenderThread

- 主线程处于 Sleep 状态，等待 Vsync 信号
- Vsync 信号到来，主线程被唤醒，Choreographer 回调 FrameDisplayEventReceiver.onVsync 开始一帧的绘制
- 处理 App 这一帧的 Input 事件(如果有的话)
- 处理 App 这一帧的 Animation 事件(如果有的话)
- 处理 App 这一帧的 Traversal 事件(如果有的话)
- 主线程与渲染线程同步渲染数据，同步结束后，主线程结束一帧的绘制，可以继续处理下一个 Message(如果有的话，IdleHandler 如果不为空，这时候也会触发处理)，或者进入 Sleep 状态等待下一个 Vsync
- 渲染线程首先需要从 BufferQueue 里面取一个 Buffer(dequeueBuffer) , 进行数据处理之后，调用 OpenGL 相关的函数，真正地进行渲染操作，然后将这个渲染好的 Buffer 还给 BufferQueue (queueBuffer) , SurfaceFlinger 在 Vsync-SF 到了之后，将所有准备好的 Buffer 取出进行合成(这个流程在讲 SurfaceFlinger 的时候会提到)

## 硬件加速绘制
正常情况下，硬件加速是开启的，主线程的 draw 函数并没有真正的执行 drawCall ，而是把要 draw 的内容记录到 DIsplayList 里面，同步到 RenderThread 中，一旦同步完成，主线程就可以被释放出来做其他的事情，调用 syncAndDrawFrame ，通知渲染线程开始工作，RenderThread 则继续进行渲染工作。


## 主线程和渲染线程的分工
主线程负责处理进程 Message、处理 Input 事件、处理 Animation 逻辑、处理 Measure、Layout、Draw ，更新 DIsplayList ，但是不涉及 SurfaceFlinger 打交道；渲染线程负责渲染渲染相关的工作，一部分工作也是 CPU 来完成的，一部分操作是调用 OpenGL 函数来完成的

当启动硬件加速后，在 Measure、Layout、Draw 的 Draw 这个环节，Android 使用 DisplayList 进行绘制而非直接使用 CPU 绘制每一帧。DisplayList 是一系列绘制操作的记录，抽象为 RenderNode 类，这样间接的进行绘制操作的优点如下

DisplayList 可以按需多次绘制而无须同业务逻辑交互
特定的绘制操作（如 translation， scale 等）可以作用于整个 DisplayList 而无须重新分发绘制操作
当知晓了所有绘制操作后，可以针对其进行优化：例如，所有的文本可以一起进行绘制一次
可以将对 DisplayList 的处理转移至另一个线程（也就是 RenderThread）
主线程在 sync 结束后可以处理其他的 Message，而不用等待 RenderThread 结束

## 布局优化建议
- 尽量多使用RelativeLayout和LinearLayout
- 将可复用的组件抽取出来并通过include标签使用
- 使用ViewStub标签来加载一些不常用的布局，其inflate性能好于SetVisiblity
- 使用merge标签减少布局的嵌套层次
- 内嵌使用包含layout_weight属性的LinearLayout会在绘制时花费昂贵的系统资源，因为每一个子组件都需要被测量两次
- 使得Layout宽而浅，而不是窄而深

