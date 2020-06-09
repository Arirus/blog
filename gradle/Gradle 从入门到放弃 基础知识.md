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


如果想指定子模块的位置，可以使用 project 方法获取 Project 对象，设置其 projectDir 参数
```
include ':app'
project(':app').projectDir = new File('./app')
```

### rootproject/build.gradle

整体项目的配置，这个是整个gradle项目编译最开始的地方。

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



