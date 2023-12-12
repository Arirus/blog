## 结构
include ':annotations'  注解层  定义ServiceProvider注解 无额外依赖
include ':registry'  定义了 ServiceRegistry 用来作为 stub ，方便loader在编译时期进行依赖  无额外依赖。生成的ServiceRegistry 维护了一个hashmap key 是 ServiceProvider 声明的接口，value 是 所有实现的组成的set，同时还维护了一个hashmap，key是 实现类，value 是一个callable的接口，返回一个实现类的对象
include ':loader'      依赖上面两个，定义ServiceLoader，维护 Class<S> mService 和  Set<S> mProviders，一个service 对应了多个 provider，加载的时候 传入class 作为service，然后去 ServiceRegistry 来获取对应的 mProviders。其是以 compileOnly project(':registry') 来引用 不会加载stub类。
include ':plugin'    定一个了一个 plugin 主要是三个操作，1.定义一个task，获取 javaCompile 的classPath等，作为输入路径，来根据注解生成java文件 2.定义compileGeneratedTask，依赖于 javaComplie，编译生成的java文件 3.将 bundle，assemble install 等依赖compile方法


## 优化
将java文件的生成task修改为增量task NewIncrementalTask， @Incremental 修饰inputFiles，同时 定义的 @TaskAction 方法 接受一个InputChanges 用来进行每次的变更修改。对每次编译后的文件节点进行缓存 生成 spiElements 文件，核心原理类似 Drouter的增量编译机制。