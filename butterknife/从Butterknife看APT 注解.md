---
date : 2018-12-22T14:03:00+08:00
tags : ["Butterknife", "APT", "注解"]
title : "从Butterknife看APT 注解"

---

# 写在最前面

如果说要评选 Android 对于新人最友好的第三方库，我觉得 Butterknife 如果称第二的话，没人敢称第一。其友好在于两点：第一是，面向的功能简单，初衷就是为了替代 `findViewByid` 方法，使得更关于逻辑上的开发；第二是，使用简单，一个注解就将 View 与资源文件元素绑定了起来。我们知道，这种越接近“底层”的方法，就越难模拟，所以 Butterknife 的技术使用可以一点不简单，那么这个系列我们就来看看，对于 Butterknife 它的技术栈都用了哪些技术。

<!--more-->

# 注解
注解是 Butterknife 中，最表象的一个技术。本篇中，我们来看看注解是怎样的一个逻辑。

注解，大家都了解，Android 开发中最常见的注解就是 `@Overrider` 当你复写父类的方法时，会在方法前面加上这个注解。用来表明这是一个覆盖了父类的方法，那么这里我们来看看 `@Overrider` 里面是怎样的逻辑。

## 元注解
```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.SOURCE)
public @interface Override {
}
```
我们看到，Overrider 上面依然被两个注解来修饰，这种修饰注解的注解被称为元注解。上面两个注解 `Target` `Retention` 是两个最常见元注解，也是我们在定义一个注解时会设定的注解。

### Target
Target 元注解主要是说明，这个注解“用于修饰的对象是什么类型的”。这里类型，是 enum 类型 ElementType 的元素。
```java
public enum ElementType {
    //“类型” 类型 如 class interface enum
    TYPE,
    //“变量” 类型
    FIELD,
    //“方法” 类型
    METHOD,
    //“参数” 类型
    PARAMETER,
    //“构造函数” 类型
    CONSTRUCTOR,
    //“局部变量” 类型
    LOCAL_VARIABLE,
    //“注解” 类型
    ANNOTATION_TYPE,
    //“报名” 类型
    PACKAGE,
}
```
例如 Override 是 `@Target(ElementType.METHOD)` 这个就是说明 Override 用于修饰方法，不能修饰其他类型。有如 BindView是 `@Target(ElementType.FIELD)` 因此 BindView 用于修饰类内的变量。当然 Target 参数可以是个数据支持多 target。默认所有都支持。

### Retention
Retention 元注解主要是说明，“注解的可见性”。Retention 的可见性分为三类：
```java
public enum RetentionPolicy {
    //仅在源码中生效
    SOURCE,
    //记录到 class 文件中
    CLASS,
    //运行时有效
    RUNTIME
}
```
其中，CLASS 是默认设置。如果有过逆向 App 的经验话，逆向后的代码依然会有 Override 修饰方法，这就是因为 Override “可见性”就是 CLASS 的。RUNTIME 是为了在运行时，来获得注解，进而通过反射来做一些事情。例如 BindView 的“可见性”就是 RUNTIME 所以，Butterknife 可以在运行时干一些事情。

## 注解参数

对于大部分运行时注解，都会需要参数，因此需要设置参数，注解中设置参数以如下方式：
```java
@Retention(RUNTIME) @Target(FIELD)
public @interface BindView {
  /** View ID to which the field will be bound. */
  @IdRes int value();
}
```
这样，我们在调用的时候，传入资源 id 就可以了：
```java
@BindView(value = R.id.btn) Button mButton;
```
对于只有一个参数的情况，可以省去 key ，直接 @BindView(R.id.btn) 如果有两个以上的参数，就需要全部说明。

对于参数，我们也可以提供默认值
```java
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface TestAnno {
    public int id() default -1;
    public String name() default "";

}
```
这样使用时可以使用默认值，这里我们就不给出例子了。

# 自定义注解

除了系统 SDK 提供，第三方库提供，我们还可以自定义注解。例如：
```java
@IntDef({ iOS, Android, unknown })

@Retention(SOURCE)
@Target({FIELD,PARAMETER})
public @interface Platform {
  int iOS = 0;
  int Android = 1;
  int unknown = 2;
}

```
有些涉及到多端的功能，可能会需要分清楚运行平台，这时候使用注解来对相应的平台进行修饰是个很好的选择。这里我们定义的了一个 Platform 的注解，用这个注解来修饰变量和参数：
```java
@Platform int mPlatform = Platform.Android;
...
public void onPlatform(@Platform int platform){
    //todo
}
```
这样表现的 platform 的含义会非常清楚。

# 小结
本篇主要就是从定义注解入手，分析了注解，元注解等基本概念，最后给出了一种在 Android 中，常使用注解的一种方式，下篇见。


