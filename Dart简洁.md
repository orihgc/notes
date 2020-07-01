# Dart特性

- 同时支持JIT和AOT
  - JIT在运行时即时编译，在开发周期中使用，可以动态下发和执行代码，开发测试效率高，但运行速度和执行性能会因为运行时即时编译受到影响。
  - AOT即提前编译，可以生成被直接执行的二进制代码，运行速度快、执行性能好，但每次执行前都需要提前编译，开发测试效率低。
  - 在开发期使用JIT编译，Flutter的热重载功能就是基于此，可以缩短产品的开发周期。
  - 在发布期使用AOT提前编译，就不需要像ReactNative那样在Js和native代码之间建立低效的方法调用映射关系。所以Dart具有运行速度快，执行性能好的特点

# 函数

- var、Object、dunamic的区别
  - var是C# 3中引入的，其实它仅仅只是一个语法. var本身并不是一种类型, 其它两者object和dynamic是类型。var声明的变量在赋值的那一刻，就已经决定了它是什么类型。
  - object之所以能够被赋值为任意类型的原因，其实都知道，因为所有的类型都派生自object. 
  - dynamic不是在编译时候确定实际类型的, 而是在运行时。
- 不指定返回值类型
  - 则返回值默认为Object
  - 若没有return，则返回null
- 如果函数中只有一个表达式，可以使用=>快速写法
- 可选参数
  - 可选的命名参数的声明使用“{}”，使用“：”指定默认值，可选的位置函数的声明使用“[]”，使用“=”指定默认值。
- 重载
  - Dart不支持方法重载
  - 使用命名构造函数
- 闭包
  - 闭包就是函数对象

# 类

- 实例变量
  
- 使用?.来确认前操作数不为空，用来代替.避免左操作室为null引发的异常
  
- 构造函数

  - 默认有构造函数，默认构造函数没有参数，子类不能继承父类的构造函数

    - 如果父类没有默认构造函数，必须手动调用

    - 语法糖：`构造函数(this.参数);

  - 命名构造函数：由此可以为类提供多个构造函数

  - 重定向构造函数

    - 重定向构造函数的主体为空**，**构造函数调用出现在冒号( `:`)之后

  - 常量构造函数

    ```dart
    final num x,y;
    const 构造函数(this.x,this.y);
    ```

  - 工厂构造函数

    - 在实现一个构造函数时使用`factory`关键字，该构造函数并不总是创建其类的新实例。例如，工厂构造函数可能会从缓存中返回实例，也可能会返回子类型的实例。

    ```dart
    class Logger {
      final String name;
      bool mute = false;
    
     // _cache是私有变量
     //_在名称前，表示该变量为私有
      static final Map<String, Logger> _cache = <String, Logger>{};
    
      factory Logger(String name) {
        if (_cache.containsKey(name)) {
          return _cache[name];
         } else {
           final logger = Logger._internal(name);
           _cache[name] = logger;
           return logger;
        }
      }
    
      Logger._internal(this.name);
    void log(String msg) {
        if (!mute) print(msg);
      }
    }
    ```

- 方法

  - Getter和Setter

- 抽象类和接口

  -  每个类都是都是隐式的接口，包括类的方法和属性。如果你想创建一个类A不继承B的实现，可以实现B的接口来创建类A。一个类允许通过`implements` 关键词可以实现多个接口

- 继承

  - 使用extends创建子类，super引用父类，子类可以重写实例方法、getter和setter，使用@override注释重写，使用@proxy注释来忽略警告

- 类变量和类方法

  - 为了常用或广泛使用的使用程序和功能，考虑使用顶层函数，而不是静态方法

- 访问控制

  - 默认类中的所有属性和方法是public的。在dart中，可以在属性和方法名前添加“_”使私有化。现在让我们使name属性私有化。

- 枚举类型

  - 枚举类型，通常称为枚举，是一种特殊类型的类，用于表示固定数量的常量值。

    `enum Color { red, green, blue }`

# 泛型

Dart是一种可选的类型语言，Dart的集合默认是异构的，单个Dart的集合可以托管各种类型的值。泛型的使用强制限制集合可以包含的数据类型，这种集合称之为类型安全的集合

- 泛型是类型安全的，写法比硬编码指定类型高效的多

- 减少重复的代码

- 例子

  ```dart
  Queue<int> queue = new Queue<int>(); 
  List <String> logTypes = new List <String>(); 
  ```

- 限制参数的类型范围，使用extends

  ```dart
  <T extends SomeBaseClass>
  ```

# 元数据

- 描述其他数据的数据，在Dart中，元数据是以@开始的修饰符

- 在@后面接着编译时的常量或调用一个常量构造函数

  @override 重写

  @proxy 代理

  @required 标记一个参数，表示该参数必须要传值

- 定义自己的元数据

  通过`library`来定义一个库，在库中定义一个相同名字的`class`，然后在类中定义`const` 构造方法。　

# 异步

- 异步模型

  - 简单说就是在某个单线程中存在一个事件循环和一个事件队列，事件循环不断的从事件队列中取出事件来执行。每当遇到耗时的事件时，事件循环不会停下来等待结果，会跳过耗时事件，继续执行其后的事件。
  - 基于事件的异步模型，只适合I/O密集型的耗时操作，因为IO耗时操作，往往是把时间浪费在等待对方传送数据或者返回结果，因此这种模型一般用于网络服务器并发

- Dart的事件循环

  1. 先查看MicroTask队列是否为空，不是则先执行MicroTask队列
  2. 一个MicroTask执行完后，检查有没有下一个MicroTask，直到MicroTask队列为空，才去执行Event队列
  3. 在Evnet 队列取出一个事件处理完后，再次返回第一步，去检查MicroTask队列是否为空

- 添加任务

  - 添加到MicroTask

  ![image-20200421144127386](/Users/bytedance/Library/Application Support/typora-user-images/image-20200421144127386.png)

  - 添加任务到Event队列

    ![image-20200421144308334](/Users/bytedance/Library/Application Support/typora-user-images/image-20200421144308334.png)

- 延时任务

  ![image-20200421144454599](/Users/bytedance/Library/Application Support/typora-user-images/image-20200421144454599.png)

  从结果可以看出，**delayed方法**调用在前面，但是它显然并未**直接将任务加入Event队列**，而是需要等待1秒之后才会去将任务加入，但在这1秒之间，后面的new Future代码直接将一个耗时任务加入到了Event队列，这就直接导致写在前面的delayed任务在1秒后只能被加入到耗时任务之后，只有当前面耗时任务完成后，它才有机会得到执行。这种机制使得延迟任务变得不太可靠，你无法确定延迟任务到底在延迟多久之后被执行。

- Future详解

  Future类是对未来结果的一个代理，封装了该任务的执行状态

  - `Future()`

  - `Future.microtask()`

  - `Future.sync()`

    同步方法，任务会被立即执行

  - `Future.value()`

  - `Future.delayed()`

  - `Future.error()`

  - `Future.then()`

    当Future中的任务完成后，我们往往需要一个回调，这个回调立即执行，不会被添加到事件队列中

  - 还可以用catchError来处理异常

    ![image-20200421145244133](/Users/bytedance/Library/Application Support/typora-user-images/image-20200421145244133.png)

  - 还可以使用静态方法wait等待多个任务全部返回后回调

    ![image-20200421145325582](/Users/bytedance/Library/Application Support/typora-user-images/image-20200421145325582.png)

- async和await

  有了这两个关键字，可以不需要调用Future相关的API，允许我们写同步代码一样写异步代码和不需要使用Future接口

  - 将async关键字作为方法声明的后缀时

    - 被修饰的方法会将一个Future对象作为返回值
    - 该方法会同步执行其中的方法直到第一个await关键字，然后暂停该方法其他部分的执行
    - 一旦由await关键字引用的Future任务执行完成，await的下一行代码将立即执行

  - 总结

    ![image-20200421150111073](/Users/bytedance/Library/Application Support/typora-user-images/image-20200421150111073.png)

- Isolate

  - 与线程最大的区别是不能共享内存，因此也不存在锁竞争的问题

  - 两个Isolate完全是两条独立的执行线，每个Isolate都有自己的事件循环，它们之间只能通过发送消息通信

  - isolate本身是隔离的意思，有自己的内存和单线程控制的实体，因为isolate之间的存在逻辑是隔离的，isolate的代码是顺序执行的。一个`Dart`程序是在`Main isolate`的Main函数开始，我们平时开发中，默认环境就是`Main isolate`，App的启动入口`main`函数就是一个`isolate`，在Main函数结束后，`Main isolate`线程开始一个一个处理`Event Queue`中的每一个`Event`。

  - 从主**`Isolate`**创建一个新的**`Isolate`**有两种方法

     `static` `Future spawnUri()`

    - spawnUri方法有三个必须的参数，
      1. 第一个是Uri，指定一个新Isolate代码文件的路径，
            第二个是参数列表，类型是List<String>，
            第三个是动态消息。

    - 需要注意，用于运行新Isolate的代码文件中，必须包含一个main函数，它是新Isolate的入口方法，该main函数中的args参数列表，正对应spawnUri中的第二个参数。如不需要向新Isolate中传参数，该参数可传空List

    
    
    　　两个Isolate是通过两对Port对象通信，一对Port分别由用于接收消息的ReceivePort对象，和用于发送消息的SendPort对象构成。其中SendPort对象不用单独创建，它已经包含在ReceivePort对象之中。一对Port对象只能单向发消息
    
  - spawn
  
     `static` `Future spawn()`
  
     除了使用spawnUri，更常用的是使用spawn方法来创建新的Isolate，我们通常希望将新创建的Isolate代码和main Isolate代码写在同一个文件，且不希望出现两个main函数，而是将指定的耗时函数运行在新的Isolate，这样做有利于代码的组织和代码的复用。spawn方法有两个必须的参数，第一个是需要运行在新Isolate的耗时函数，第二个是动态消息，该参数通常用于传送主Isolate的SendPort对象。
  
     无论是上面的**spawn**还是**spawnUri**，运行后都会创建两个进程，一个是主Isolate的进程，一个是新Isolate的进程，两个进程都双向绑定了消息通信的通道，即使新的**Isolate**中的任务完成了，它的进程也不会立刻退出，因此，当使用完自己创建的**Isolate**后，最好调用**newIsolate.kill(priority: Isolate.immediate)**;将Isolate立即杀死。
  
- Flutter中创建isolate

  - compute函数必须有两个必须的参数
    - 第一个是待执行的函数，这个函数必须是一个顶级函数，不能是类的实例方法，可以是类的静态方法，
    - 第二个参数为动态的消息类型，可以是被运行函数的参数。

- 使用场景

  - 根据任务的平均时间来确定使用是使用Isolate还是使用Future
    - 方法执行在几毫秒或十几毫秒，则使用Future
    - 如果一个任务在几百毫秒或以上，则使用Isolate
      - Json解码
      - 加密
      - 图形处理：裁剪
      - 网络请求：加载资源、图片