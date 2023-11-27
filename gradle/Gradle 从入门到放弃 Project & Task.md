# Project & Task

Gradle 体系下每一个 build.gradle 文件对应的就是一个 Project。无论是RootProject还是SubProject。

官方文档 有 [Project API](https://docs.gradle.org/current/javadoc/org/gradle/api/Project.html) 所有的详细调用。

Project 本身继承于 ExtensionAware PluginAware。其中前者控制着所有的 extension 内容，后者则是控制所有的 plugins 内容。
因此 Project 相关API分为三部分，一个是 Project 自己的，还有另外两部分（extension 和 plugins）相关设置。

## Project

### Project API
多是 Project Task dependency 配置等相关内容，这里只是列举出常用的API，详细内容还是参考官方API
```java
//其本身以及子project的相关回调，可以在此处 做全局的统一配置，函数调用等。和直接写在 build.gradle 文件中的效果是一样的。
void allprojects(Closure configureClosure);

//子project的相关回调，可以在此处 做全局的统一配置，函数调用等。和直接写在 build.gradle 文件中的效果是一样的。
void subprojects(Closure configureClosure);

// 来创建一个 Configuration
void configurations(Closure configureClosure);

// 返回ConfigurationContainer 可以全局修改 配置
ConfigurationContainer getConfigurations();

// 为当前工程配置脚本工程
void buildscript(Closure configureClosure);

//除此之外还有 文件等相关操作
```

### ExtensionAware API
用来实现运行时的 extension 的实例。其方法中仅仅就一个方法，返回 ExtensionContainer 对象，可以通过这个对象进行 增加 创建 查找等相关操作。

```java
// 增加一个新的 extension 到 ExtensionContainer 中
<T> void add(Class<T> publicType, String name, T extension);

// 创建并新增一个 extension 到 ExtensionContainer 中
<T> T create(String name, Class<T> type, Object... constructionArguments);

// 查找给定Type类型的 extension
<T> T findByType(Class<T> type);
```


### PluginAware API
就是用来添加 或者获取插件的。
```java
// 添加一个新的插件
void apply(Closure closure);

// 获取插件的 Container，从而来获取插件
PluginContainer getPlugins();
```

## 拓展属性
我们可以通过以下两种方式来定义扩展属性：
- 通过 ext 关键字定义扩展属性
- 在 gradle.properties 下定义扩展属性

```java
//当前在根 build.gradle 下
//方式1：ext.属性名
ext.test = 'test'

//方式2：ext 后面接上一个闭包
ext{
  test1 = 'test'
}

//方式3:单独一个文件来存放 ext 配置，然后导入需要引入的地方
apply from: "$rootDir/version.gradle"

ext {
    agp_version = "4.2.0"
    kotlin_version = "1.5.32"
}
```

## Task

Task 中文翻译即任务，它是 Gradle 中的一个接口，代表了要执行的任务，不同的插件可以添加不同的 Task，每一个 Task 都要和 Project关联。gradle 执行的每个命令 都是一个task，包括app打包也是一样的。

我们在定义一个task的时候，会发现task在创建的时候，闭包里的命令也是会执行的。
```java
task("CustomTask"){
    doLast {
        println("CustomTask doLast 闭包内")
    }
    println("CustomTask 闭包内")
    doFirst {
        println("CustomTask doFirst 闭包内")
    }
}
```
只有当执行
```shell
./gradlew CustomTask
```
才会将 doLast doFirst 执行

### Task类型 
Task 默认类型为 DefaultTask，基本上我们在自定义的时候 都会是继承 DefaultTask，除此之外还有 Copy，Delete 等 Task。

### TaskContainer
TaskContainer 为task的整体容器，我们创建的 task 最终也会被加入到 TaskContainer。
```java
//查找task
findByPath(path: String): Task 
getByPath(path: String): Task
getByName(name: String): Task

//创建task
create(name: String): Task
create(name: String, configure: Closure): Task 
```

### Task执行顺序
task 可以被指定执行顺序。如果没有显式依赖的话，则task是孤立的。
可以通过 
- dependsOn 强依赖方式可细分为静态依赖和动态依赖
  - 静态依赖：在创建 Task 的时候，直接通过 dependsOn 指定需要依赖的 Task
  - 动态依赖：在创建 Task 的时候，不知道需要依赖哪些 Task，需通过 dependsOn 动态依赖符合条件的 Task
- mustRunAfter：指定必须在哪个 Task 执行完成之后执行
- shouldRunAfter：跟 mustRunAfter 类似，区别在于不强制，不常用
- finalizeBy：在当前 Task 执行完成之后，指定执行的 Task


