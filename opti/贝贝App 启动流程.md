贝贝使用了 Tinker，因此所有的优化操作的起点是从 `BeiBeiApplicationLike` 的 onCreate 方法开始的。

使用了 `AppLaunchManager` 来进行各个模块的启动的。主要是分为这几个部分进行观察的。

- Application onCreate() 模块
- SplashActivity onCreate() 模块
- SplashActivity enterApp() 模块
- HomeActivity onCreate() 模块
- HomeActivity onCreate() 懒加载模块
- HomeActivity onResume() 懒加载模块

## Application onCreate()
直接启动模块 没有懒加载模式。这里按照主要顺序主要是做了这些工作：
- 打点 app onCreate 开始时间
- 载入本地so文件（为了获取相应密钥和验证App）
- 配置 App 相关项 (Bugly、 报名、 App Scheme等)
- 在主进程开始之前
    - 熔断机制加载，与处理(清理缓存 config配置等)
    - 初始化 sessionId，用来追逐一次 App 从启动到结束的完整流程
    - 初始化 HttpDns ，并预初始化核心域名（请求过程是异步的）
- 通用进程中    
    - 初始化 UDID 生成器
    - CrashHandler 设置，用于收集崩溃信息
    - 配置 Https 的开关，判断是否强制开启
    - 自动更新功能初始化
    - 设置设备配置（UDID 应用渠道）
    - 应用注册 生命回调 监听器
    - Toolbar 通用配置
    - 扫一扫通用配置

    - 初始化网络状态管理器
    - 配置网络相关log
    - 注册网络状态变化监听器
    - 网络初始化设置（HttpDns 拦截器配置 统计上报）
    - BaseApiRequest 通用配置
    
    - 账户管理器配置
    - 分享初始化配置

    - imageLoader 配置
    - 打点与统计 配置
    - 火眼上传 配置

    - 下拉刷新控件初始化

    - 初始化 HBRouter
- 在主进程中
    - msgChannel 初始化
    - LayoutInflater  设置 Delegate
    - 初始化 ActionManager
    - 注册广告系统
    - 调用 所有 app_create的所有action（目前只有设置二维码页面的启动动画和创建上传图片文件夹两个操作）
    - ActivityStackManager 初始化
    - 初始化 LoadingView 的样式
    - HBLeaf 初始化
    - 风控打点初始化
    - Hybird Log 系统初始化

- 初始化日志系统 (通用日志系统，glide日志系统，Dns日志系统等)
- Tinker 初始化
- 验证 App 签名是否被篡改等。
- 打点 app onCreate 结束时间

至此 Application 的 onCreate 部分全部完成。

## SplashActivity onCreate() 模块
此模块没做什么事情主要就是：
- 检测当前的网络状态
- 重置闹钟管理器

## SplashActivity enterApp() 模块
- MTA 初始化
- Bugly 初始化
- 首页广告预加载
- 皮肤氛围预加载
- 推送初始化

## HomeActivity onCreate() 模块
- 首页弹窗初始化
- 清除网络缓存

## HomeActivity Lazy onCreate() 模块
- 检查 hotfix
- 检查 是否升级

## HomeActivity Lazy onResume() 模块
- 延时两秒上传启动时间
