# Jetpack Navigation 

这个库主要是用来 Fragment 之间的跳转，但本质上不算是个路由库，只是一个导航。

我们常用的单Activity多Fragment模式下，需要通过 FragmentManager 来对 Fragment 来进行管理。而 Navigation 就是简化这些操作。

## 引入
```java
dependencies {
  def nav_version = "2.3.5"

  // Java language implementation
  implementation "androidx.navigation:navigation-fragment:$nav_version"
  implementation "androidx.navigation:navigation-ui:$nav_version"

  // Kotlin
  implementation "androidx.navigation:navigation-fragment-ktx:$nav_version"
  implementation "androidx.navigation:navigation-ui-ktx:$nav_version"

  // Feature module Support
  implementation "androidx.navigation:navigation-dynamic-features-fragment:$nav_version"

  // Jetpack Compose Integration
  implementation "androidx.navigation:navigation-compose:1.0.0-alpha10"
}
```

## 使用
