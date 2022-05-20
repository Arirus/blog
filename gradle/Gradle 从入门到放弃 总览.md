# 写在最前面

Gradle 最开始接触是刚开始搞 Android 的时候。那时候每次从 github 上clone一个新工程，每次都要编译好长时间。之后，我学会了每次都修改 gradleWrapper 文件，compileVersion 等操作，对于 gradle 的了解也仅仅于此。

之后慢慢对这里了解也是2、3年之后，



Gradle的plugins{}和apply plugin的区别：

plugins{}块这种方式引入的插件来自Gradle官方插件库；

使用“buildscript {}”块指定第三方库作为Gradle插件的话，指定插件就需要使用“apply plugin”了。


Plugin 和 Extension 的区别

android 打包配置，就是 Gradle 的 Extension，翻译成中文意思就叫扩展。它的作用就是通过实现自定义的 Extension，可以在 Gradle 脚本中增加类似 android 这样命名空间的配置，Gradle 可以识别这种配置，并读取里面的配置内容。

Plugin 就是自定义的插件。插件读取 Extension



