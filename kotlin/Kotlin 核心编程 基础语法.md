# Kotlin 核心编程 基础语法

## 变量声明

- 对于变量无需声明类型，可以自己推倒
- 函数的返回类型，也可以自己推倒
- val 与 var 区别

## 高阶函数和Lambda

### 函数类型

(int) -> Unit

- 通过>符号来组织参数类型和返回值类型，左边是参数类型，右边是返回值类型；
- 必须用一个括号来包裹参数类型；
- 返回值类型即使是Unit，也必须显式声明。


(int) -> ((int) -> Unit) 

返回的类型也是一个函数类型

### 方法和成员引用

Kotlin存在一种特殊的语法，通过两个冒号来实现对于某个类的方法进行引用。

```java
str::length  //引用一个 String 对象的length方法
```

### Lambda

- 一个 Lambda必须用 {} 来包裹
- 如果Lambda声明了参数部分的类型，且返回值类型支持类型推导，那么Lambda变量就可以省略函数类型声明； val sum:(Int,Int) > Int={ x,y > x+y }
- 如果Lambda变量声明了函数类型，那么Lambda的参数部分的类型就可以省略。val sum={ x:Int,y:Int > x+y }
- 如果Lambda返回的不是Unit，那么默认最后一行表达式类型就是返回类型

### 函数 Lambda 和 闭包

- fun在没有等号、只有花括号的情况下，是我们最常见的代码块函数体，如果返回非Unit值，必须带return。
- fun带有等号，是单表达式函数体。该情况下可以省略return。 fun foo(x:Int,y:Int) = x+y
- 不管是用val还是fun，如果是等号加花括号的语法，那么构建的就是一个Lambda表达式，Lambda的参数在花括号内部声明.
    val foo={ x:Int,y:Int > x+y }   //  foo.invoke(1,2)或foo(1,2)
    fun foo(x:Int) = { y:Int > x+y } // foo(1).invoke(2)或foo(1)(2)


一个闭包可以被当作参数传递或者直接使用，它可以简单地看成“访问外部环境变量的函数”。Lambda是Kotlin中最常见的闭包形式。

fun foo(x:Int) = { y:Int > x+y } 其实是等价于

```java
fun foo(x:Int):(Int) -> Int{
    return {y:Int -> x+y}
}
```



