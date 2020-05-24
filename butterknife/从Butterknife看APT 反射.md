---
date : 2018-12-22T14:03:00+08:00
tags : ["Butterknife", "APT", "反射"]
title : "从Butterknife看APT 反射"

---

# 写在最前面
上一篇，我们分析了注解的一些基本概念，本篇来看看常常和注解一起出现的反射，Butterknife 中就使用了部分反射的方法来实现了某些构造函数的调用。那么我们先看看反射的基本用法，再看看 Butterknife 中，反射的使用方式。

# 反射
简单的来说，反射提供了一个思路，就是从类方面下手，来获取或者执行相应的操作，而不是从对象下手。可以在运行时获得程序或程序集中每一个类型的成员和成员的信息。

**反射的核心是 JVM 在运行时才动态加载类或调用方法/访问属性**

## 基本使用

### 获取 Class 对象
Class 对象，差不多是反射机制的入口了，常用三种方法来获取一个 Class 对象：
```java
public static Class<?> forName(String className) //通过名字来获取 Class，主要类名要包含完整路径。
Class<?> clazz = Activity.class; // 直接调用 .class 方法来获取 Class
Class<?> clazz = str.getClass(); // 调用实例的 getClass 方法
```
三种方法都能获得一个 Class 对象。

### 获取 Field 对象 