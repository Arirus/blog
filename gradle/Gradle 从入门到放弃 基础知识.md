# 基础知识

## Gradle 是什么

简单来说是基于 groovy 来编写的自动化编程框架。里面包含很多task，当我们在打包的时候，会有javac编译，dex打包，apk打包，这每一步操作都是由一个gradle.task来完成的。

编写语言是使用的 groovy，这个语言的特性，本系列就不详细描述了，需要了解其方法调用和闭包的概念即可。

## 目录结构

这个大家可能已经非常熟悉了，就不放截图了，一个树状结构再看看好了：
```
.                                //根目录
├── app                          // subProject :app
│   ├── app.iml
│   ├── build
│   ├── build.gradle             // :app 的 build.gradle 脚本
│   ├── libs
│   ├── proguard-rules.pro
│   └── src
│       ├── androidTest
│       ├── main
│       └── test
├── build
├── build.gradle                 // rootProject 的 build.gradle 脚本
├── gradle                      
│   └── wrapper                  // gradle wrapper 的相关信息
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties   //wrapper 的版本
├── gradle.properties            // 设置的属性，可以是JVM的，也可以是别的
├── gradlew                      // 执行脚本
├── gradlew.bat
├── local.properties             // 可以不了解
└── settings.gradle              // gradle Project 设置文件
```

内容不再细说，这里需要强调几个概念：
- gradle 在构建时，有个全局的 gradle 对象，在所有的 project 中都能找到。
- 每一个 build.gradle 文件都是会生成一个 project 对象
- settings 文件会生成一个 settings 对象

### settings.gradle

settings.gradle 是负责配置项目的脚本，我们通过它来支持多少个项目一起编译。
常用方法是有这些
- include(projectPaths)          //给定的project加到build 列表中，不过都子一级别的project
- includeFlat(projectNames)      //给定的project加到build 列表中，不过都兄弟级别的project
- project(projectDir)            //跟进一个给定路径返回project
- includeBuild(project) // 用于外部库作为依赖添加进来


如果想指定子模块的位置，可以使用 project 方法获取 Project 对象，设置其 projectDir 参数
```
include ':app'
project(':app').projectDir = new File('./app')
```

settings.gradle  文件负责管理各个 subProjects，本身并不是 project ，因此没有 tasks

kts 这里貌似有bug，需要 standalone script 才可以正常识别kotlin 相关语法

### rootproject/build.gradle

整体项目的配置，这个是整个gradle项目编译最开始的地方。最终编译成一个 Project 对象。

其中几个主要方法有：

- buildscript // 配置脚本的 classpath
- allprojects // 配置项目及其子项目
- respositories // 配置仓库地址，后面的依赖都会去这里配置的地址查找
- dependencies // 配置项目的依赖

最常见的写法：
```
buildscript {
    repositories {
        google()
        jcenter()
        
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:3.5.3'
        
        // NOTE: Do not place your application dependencies here; they belong
        // in the individual module build.gradle files
    }
}

allprojects {
    repositories {
        google()
        jcenter()
        
    }
}
```
### module/build.gradle

build.gradle 是子项目的配置，对应的也是 Project 类。
子项目和根项目的配置是差不多的，不过在子项目里可以看到有一个明显的区别，就是引用了一个插件 apply plugin "com.android.application"，后面的 android dsl 就是 application 插件的 extension。

### 依赖

#### implementation
依赖项在编译时对模块可用，并且仅在运行时对模块的消费者可用。 通常用于业务层模块。

#### api
依赖项在编译时对模块可用，并且在编译时和运行时还对模块的消费者可用。 通常用在底层SDK。

## init.gradle
在 gradle 里，有一种 init.gradle 比较特殊，这种脚本会在每个项目 build 之前先被调用，可以在其中做一些整体的初始化操作，比如配置 log 输出等等。

init.gradle 文件放在 `USER_HOME/.gradle/`目录下，这样初始化的时候会首先初始化到这里。

## 生命周期

### 初始化阶段
初始化阶段主要做的事情是有哪些项目需要被构建，然后为对应的项目创建 Project 对象
到此阶段，settings 读取完成，知道有多少子项目参与构建。

### 配置阶段
配置阶段主要做的事情是对上一步创建的项目进行配置，这时候会执行 build.gradle 脚本，并且会生成要执行的 task。
到此阶段，会生产task的有向无环图。


### 执行阶段
执行阶段主要做的事情就是执行 task，进行主要的构建工作。

### 流程图

```
.                                //根目录
├── app                          // app
│   ├── build
│   └──build.gradle             
├── build
├── build.gradle                 // rootProject 的 build.
├── buildSrc                     // buildSrc 库
│   ├── build
│   └──build.gradle
├── plugin                      // plugin 库
│   ├── build
│   └──build.gradle
└── settings.gradle              // 根Setting
```
在 setting 中，和4个build.gradle文件中配置完log

``` java
// setings 文件初始化
*** settings settingsEvaluated settings 'Plugin'
*** settings projectsLoaded build 'Plugin'

// buildSrc configuration 
> Configure project :buildSrc
// buildSrc 执行
*** buildSrc afterProject  project ':buildSrc'
*** buildSrc afterEvaluate  project ':buildSrc'

> Task :buildSrc:compileKotlin UP-TO-DATE
...
> Task :buildSrc:build UP-TO-DATE

// rootProject configuration 
> Configure project :
*** rootBuild afterProject root project 'Plugin'
*** rootBuild afterEvaluate root project 'Plugin'

// app configuration 
> Configure project :app
*** rootBuild beforeProject project ':app'
*** rootBuild beforeEvaluate project ':app'
// android extension 开始
*** Android Extension start
*** rootBuild afterProject project ':app'
*** rootBuild afterEvaluate project ':app'

// plugin configuration 
> Configure project :plugin
*** rootBuild beforeProject project ':plugin'
*** rootBuild beforeEvaluate project ':plugin'


*** rootBuild afterProject project ':plugin'
*** rootBuild afterEvaluate project ':plugin'
*** settings projectsEvaluated build 'Plugin'
// 完成
```

基本流程简单来说：
初始化阶段：
    setting.gradle 执行，settingsEvaluated 和 projectsLoaded 回调收到。
**注意：**优先 buildSrc ，afterProject 和 afterEvaluate 回调收到。buildSrc 会直接进行构建，进行build。
配置阶段：
    先进行根工程的 beforeProject beforeEvaluate 和 afterProject afterEvaluate。中间进行 build.gradel 的配置
    遍历 subProject，执行 beforeProject beforeEvaluate 和 afterProject afterEvaluate。中间进行 build.gradel 的配置
    最后 settings projectsEvaluated 收到回调。此时 taskGraph 建立完毕。
执行阶段：
    Gradle.taskGraphBeforeTask 和 Gradle.taskGraphAfterTask  收到回调。中间会进行 task.doAction 的执行。
    Gradle.buildFinish 回调


使用
```java
gradle.taskGraph.beforeTask {Task task ->
    task.doFirst {
        task.ext.beginOfTask = System.currentTimeMillis()
    }

    task.doLast {
        println "执行阶段，$task 耗时：${System.currentTimeMillis() - task.ext.beginOfTask} ms"
    }
}
打印出每个任务的执行耗时。
```


## 自定义 task

自定义task属于较为常用的功能之一，常用写法：
```
task myTask {
    println 'myTask in configuration'
    doLast {
        println 'myTask in run'
    }
}

tasks.create("mytask").doLast {
}
```

Task 的一些重要方法分类如下：

- Task 行为
    - Task.doFirst
    - Task.doLast

- Task 依赖顺序
    - Task.dependsOn
    - Task.mustRunAfter
    - Task.shouldRunAfter
    - Task.finalizedBy

- Task 是否可用
    - Task.enabled


Gradle 的基础知识差不多就是这样，在不自定义 Gradle-plugin 的前提下，目前的内容算是 Gradle 的基础知识了。