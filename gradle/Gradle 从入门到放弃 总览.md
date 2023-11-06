# 写在最前面

Gradle 最开始接触是刚开始搞 Android 的时候。那时候每次从 github 上clone一个新工程，每次都要编译好长时间。之后，我学会了每次都修改 gradleWrapper 文件，compileVersion 等操作，对于 gradle 的了解也仅仅于此。

之后慢慢对这里了解也是2、3年之后，


## *.gradle.kts API
普通的gradle脚本是 实现了一个类：
```java
KotlinBuildScript{
    @Suppress("unused")
    open fun buildscript(@Suppress("unused_parameter") block: ScriptHandlerScope.() -> Unit): Unit =
        internalError()

    /**
     * Configures the plugin dependencies for this project.
     *
     * @see [PluginDependenciesSpec]
     */
    @Suppress("unused")
    fun plugins(@Suppress("unused_parameter") block: PluginDependenciesSpecScope.() -> Unit): Unit =
        invalidPluginsCall()
}
```
这个仅仅是针对非Settings的 gradle 脚本。
```java
apply(from = "$filePath")
```
来引入该脚本，注意引入之后 该脚本无法知道 是settings的引入 还是 project 的引入 

注意：对于 ext 这个属性 ，kts 使用方法和原先不同了
```java
grovvy
定义
ext {
    springVersion = "3.1.0.RELEASE"
    emailNotification = "build@master.org"
}
使用
println springVersion


kotlin
定义
val springVersion by extra("3.1.0.RELEASE")
val emailNotification by extra { "build@master.org" }
    或者
extra["add1"] = 111
extra["add2"] = 222
使用
println(springVersion)

```
ext kts 和 groovy 最大区别就是 如果在外部调用处 获取对应的 ext.property，groovy 是可以使用定义的名字的 ，但是kts 不行
```java
project.ext { foo = "bar" }

assert project.ext.get("foo") == "bar"
assert project.ext.foo == "bar"
assert project.ext["foo"] == "bar"

assert project.foo == "bar"
assert project["foo"] == "bar"

groovy 以上assert 都是支持的 但是 kts 不行
```
ext属性可以伴随对应的ExtensionAware对象在构建的过程中被其他对象访问，例如你在rootProject中声明的ext中添加的内容，就可以在任何能获取到rootProject的地方访问这些属性，而如果只在rootProject/build.gradle中用def来声明这些变量，那么这些变量除了在这个文件里面访问之外，其他任何地方都没办法访问。

此外要注意，ext属性是属于拥有他的相应的对象的，比如Project对象，因此只要能访问到对应的Project对象，就能访问到对应的ext属性

seetings 文件中获取不到 rootProject的ext ，因此要放到 gradle 中去
```java
(gradle as ExtensionAware).extra["myObject"]
读取的时候 使用 类似 进行读取
(gradle as ExtensionAware).extra.get("myObject")
```



## settings API
settings 脚本是用来控制当前 有多少个 子Project的。其实现类是：
```java
KotlinSettingsScript{
    open fun plugins(@Suppress("unused_parameter") block: PluginDependenciesSpecScope.() -> Unit): Unit =
    @Suppress("unused")
    open fun buildscript(@Suppress("unused_parameter") block: ScriptHandlerScope.() -> Unit): Unit =
        internalError()


    apply(closure)；
    findProject(projectDir)；
    include(projectPaths)
    includeBuild(rootProject)
    includeFlat(projectNames)
    project(projectDir)
}
```

settings 文件本身也可以配置 buildscript，同时引入 apply plugin ，但是感觉一般没什么必要 也没人这么干，可以把 rootProject 的build文件 的 buildscript 放到setting里面做

settings 本身还是定位 project 管理，这里看看三种include的区别：
```java
include 主要是用于 settings 文件中，rootProject 下面各个 subPorject 的引用，也就是最最常规的用法
这里 project 可以用来生成一个project

// include two projects, 'foo' and 'foo:bar'
// directories are inferred by replacing ':' with '/'
include 'foo:bar'

// include one project whose project dir does not match the logical project path
include 'baz'
project(':baz').projectDir = file('foo/baz')

// include many projects whose project dirs do not match the logical project paths
file('subprojects').eachDir { dir ->
  include dir.name
  project(":${dir.name}").projectDir = dir
}

使用  include + project 可以实现源码与AAR的切换
```java
//步骤一：在 settings.gradle 中引入源码工程
include ':test'
project(':test').projectDir = file('当前工程的绝对路径')

//步骤二：在根 build.gradle 下进行如下配置
allprojects {
    configurations.all {
        resolutionStrategy {
            dependencySubstitution {
                substitute module("com.dream:test") with project(':test')
            }
        }
    }
}
```


includeFlat 这个将与rootProject 平行的project，include到当前的setting里面
As an example, the name a add a project with path :a, name a and project directory $rootDir/../a.
includeFlat("demoLib") 只一个名字就ok 不用前面:

这种情况下 a 这个project 就是一个library project 不能是另一个 rootProject


最后一种是 includeBuild 这个是将多个独立的rootProject进行构建
includeBuild('../TheOneSDKGlobal') {
    dependencySubstitution {
        substitute module('com.xiaoju.nova:the-one-globalsdk') with project(':TheOneSDKGlobal')
    }
}
通过这种方式，将两个工程一起编译，同时可以替换某个工程中的依赖

这种方式和上面那种方式的根本区别在于，上面这种是把 外部project 作为module依赖，但是includeBuild这种方式是同时编译两个工程 允许某个工程的子工程直接替换到另一个工程里面
```


Gradle的plugins{}和apply plugin的区别：

plugins{}块这种方式引入的插件来自Gradle官方插件库；

使用“buildscript {}”块指定第三方库作为Gradle插件的话，指定插件就需要使用“apply plugin”了。


Plugin 和 Extension 的区别

android 打包配置，就是 Gradle 的 Extension，翻译成中文意思就叫扩展。它的作用就是通过实现自定义的 Extension，可以在 Gradle 脚本中增加类似 android 这样命名空间的配置，Gradle 可以识别这种配置，并读取里面的配置内容。

Plugin 就是自定义的插件。插件读取 Extension



