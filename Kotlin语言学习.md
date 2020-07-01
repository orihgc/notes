# 变量和函数

## 变量

- 只允许在变量前声明两种关键字
  - val不可变
  - var可变
- 类型推导
  - 如果对一个变量延迟赋值，kotlin就无法推导其类型
  - 可以显示声明`var a:Int=10`
  - Kotlin完全抛弃了Java的基本数据类型，完全使用对象数据类型

## 函数

```kotlin
fun method(param1: Int): Int {}
```

- 返回值可选

- 语法糖：当函数只有一行代码的时候

  ```kotlin
  fun largerNumber(num1: Int, num2: Int): Int = max(num1,num2)
  ```

# 逻辑控制

## If

```kotlin
val value = if (num1 > num2) {
    num1
} else {
    num2
}
```

语法糖使用：

```kotlin
fun largerNumber(num1: Int, num2: Int) = if (num1 > num2) { num1 } else { num2 }
```

## When

- 精确匹配

```kotlin
fun getStore(name: String) = when (name) {
    "Tom" -> 86
    "Jack" -> 90
    "Lily" -> 100
    else -> 0
}
```

- 类型匹配

```kotlin
fun checkNumber(num: Number) {
    when (num) {
        is Int -> print("Int")
        is Double -> print("Double")
        else -> print("not support")
    }
}
```

- 不带参数的用法

```kotlin
fun getStore(name: String) = when {
    name.startsWith("Tom") -> 86
    name == "Jack" -> 90
    name == "Lily" -> 100
    else -> 0
}
```

## 循环

- 区间

```kotlin
val range = 1..10
for (i in range) {
    print(i)
}
```

- 左闭右开区间

```kotlin
val range = 0 until 10;
```

- 步距

```kotlin
for (i in range step 2) {
    print(i)
}
```

- 降序区间

```kotlin
val range = 10 downTo 0
```

# 类和对象

- 没有new关键字

## 继承

- 默认所有非抽象类不可被继承，若要继承，则需要加上open关键字
- 使用一个冒号表示继承

```kotlin
open class Person {}
class Student : Person() {}
```

## 构造函数

- 主构造函数

```kotlin
open class Person(val name: String, val age: Int) {}
class Student(val sno: String, val grade: Int, name: String, age: Int) : Person(name, age) {}
```

## 次构造函数

- 当一个类既有主构造函数又有次构造函数时，所有次构造函数都必须调用主构造函数

```kotlin
class Student(val sno: String, val grade: Int, name: String, age: Int) : Person(name, age) {//name和age不需要val和var来修饰，因为修饰后，就自动成为该类的字段，会和Person冲突
    constructor(name: String, age: Int) : this("", 0, name, age) {}
    constructor() : this("", 0);
}
```

- 类中只有次构造函数，而没有主构造函数

```kotlin
class Student : Person {
    constructor(name: String, age: Int) : super(name, age)
}
```

## 接口

```kotlin
interface Study {
    fun readBooks()
    fun doHomeWork()
}

open class Person(val name: String, val age: Int) {}

class Student(name: String, age: Int) : Person(name, age),Study {
    override fun readBooks() {}

    override fun doHomeWork() {}
}
```

- 允许对接口中的函数进行默认实现

```kotlin
interface Study {
    fun readBooks()
    fun doHomeWork(){println("做作业")}
}
```

## 数据类

- 数据类通常需要重写equals()、hashCode()、toString()这几个方法

```kotlin
data class Cellphone(val brand: String, val price: Double)
```

- data关键字表明这是一个数据类，kotlin会自动生成equals()、hashCode()、toString()

## 单例类

```kotlin
object Singleton {
    fun method() {}
}
Singleton.method()
```

# lambda表达式

## 集合的创建和遍历

- listOf()初始化集合
  - listOf创建的是一个不可变集合

```kotlin
val list = listOf("apple", "banana", "orange");
```

- 可变集合

```kotlin
val list = mutableListOf("apple", "banana", "orange");
```

- setOf和mutableSetOf
- mapOf和mutableMapOf

```kotlin
    val map = mutableMapOf("apple" to 1, "banana" to 2, "orange" to 3)
    map["pear"] = 4;
    for ((fruit,number) in map) {
        print(fruit+number)
    }
```

## 集合的函数式API

- 找到list中单词最长的那个水果

  - lambda表示一段代码，作为参数传递

    ```kotlin
    var lambda = { fruit: String -> fruit.length }
    list.maxBy(lambda)
    ```

  - 当lambda是函数的最后一个参数时，可以将lambda移到函数括号的外面

    ```kotlin
    list.maxBy(){ fruit: String -> fruit.length }
    ```

  - 当lambda是唯一一个参数时，可以将函数括号沈略

    ```kotlin
    list.maxBy(){ fruit: String -> fruit.length }
    ```

  - 由于有类型推导，可以不需要参数类型

    ```kotlin
    list.maxBy(){ fruit -> fruit.length }
    ```

  - 当lambda表达式只有一个参数时，可以直接用it来代替

    ```kotlin
    val maxLength = list.maxBy { it.length }
    ```

- 将所有水果大写，map

```kotlin
list.map{it.toUpperCase()}
```

- 过滤5个字母以内的水果

```kotlin
list.filter { it.length < 5 }
```

- any和all

```kotlin
list.any { it.length<=5 }
list.all { it.length<=5 }
```

## Java函数式API的使用

- 匿名类

```kotlin
Thread(object : Runnable {
    override fun run() {
        print("Thread is running")
    }
}).start()
```

- 若只有一个待实现的方法

```kotlin
Thread(Runnable { print("Thread is running") }).start()
```

- 若Thread方法的参数列表只有一个Java单抽象接口参数

```kotlin
Thread({ print("Thread is running") }).start()
```

- 若lambda表达式是方法的最后一个参数，则可以将lambda移到括号外面，若是方法的唯一一个参数，还可以省略括号

```kotlin
Thread { print("Thread is running") }.start()
```

# 空指针检查

- Kotlin将空指针检查提前到了编译期

- 那么如何传null呢？

  ```kotlin
  fun doStudy(study:Study?){
    if(study!=null){
      study.readBooks
    }
  }
  ```

- 判断辅助工具

  - ?.当对象为空时就啥都不做

  `study?.readBoosk()`

  - ?:如果左边表达式不为空，返回左边表达式

  `var  c=a?:b`

  - !!非空断言工具，强行通过编译

- let

  ```kotlin
  study?.let(stu->{
     stu.readBooks()
     stu.doHomework()
  })
  ```

  ```kotlin
  study?.let(
     it.readBooks()
     it.doHomework()
  })
  ```

- let可以处理全局变量判空问题，但是if判断语句则会出错

# Kotlin中的小魔术

## 内嵌表达式

&{}

## 函数的参数默认值

- 函数的参数默认值

```kotlin
fun printParams(num: Int, str: String = "hello") {}
```

- 键值对传参数

```kotlin
printParams(str = "world",num = 23)
```

# 标准函数

- let

  - 主要作用是配合?.操作符进行判空

- with

  - 接收两个参数，1是任意类型的对象，2是一个Lambda表达式

  ```kotlin
  val list = listOf("Apple", "Banana", "Orange")
  val result = with(StringBuilder()) {
      append("Start eating")
      for (fruit in list) {
          append(fruit)
      }
      append("end")
  }.toString()
  ```

- run

  ```kotlin
  val list = listOf("Apple", "Banana", "Orange")
  val result = StringBuilder().run {
      append("Start eating")
      for (fruit in list) {
          append(fruit)
      }
      append("end")
  }.toString()
  ```

- apply

  - 无法指定返回值

  ```kotlin
  val list = listOf("Apple", "Banana", "Orange")
  val result = StringBuilder().apply {
      append("Start eating")
      for (fruit in list) {
          append(fruit)
      }
      append("end")
  }.toString()
  ```

# 定义静态方法

- 静态方法一般用于工具类，在kotlin中，工具类完全可以使用单例类来实现

- companion object

  - 如果只想让某一个方法变成静态方法

  - 但其实companion object中的方法也不是静态方法，实际上是在Util类的内部创建一个伴生类，调用Util.doAction()实际上是调用了Util类中伴生对象的doAction方法

- 若确实需要定义真正的静态方法

  - 在单例类或companion object中的方法上加上@JvmStatic注解，就会将其编译成真正的静态方法
  - 顶层方法
    - 在Kotlin，所有顶层方法可以在任意位置调用
    - 在Java中，需要文件名.doSomething()的方式调用

# 延迟初始化和密封类

- lateinit

  判断是否已经完成初始化

  ```kotlin
  if(!::变量.isInitialized){}
  ```

- 密封类sealed class

# 扩展函数和运算符重载

- 扩展函数

```kotlin
fun ClassName.methodName(param1:Int,param2:Int):Int{}
```

向那个类添加扩展函数，则定义一个同名的kotlin文件

- 运算符重载

```kotlin
class Money(val value:Int){
     operator fun plus(money:Money):Money{
          val sum=value+money.value
          return Money(sum)
     }
  
     operator fun plus(newValue:Int):Money{
          val sum=value+newValue
          return Money(sum)
     }
}
```



