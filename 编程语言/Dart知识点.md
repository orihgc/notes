# 异步

## 事件队列和微任务队列

- 异步

  - 多线程的缺陷
    - 开辟线程耗费资源
    - 多个线程共享内存时需要加锁
  - 基于事件的异步模型
    - 只适合IO密集型的耗时操作
    - 计算密集型的操作，应当利用处理器的多核，实现并行计算

- Dart的两个队列

  - 微任务队列
  - 事件队列

  永远都是先检查微任务队列是否为空，为空才去执行事件队列

- 调度任务

  - 添加到微任务队列

    ```
    scheduleMicrotask(myTask)
    Future.microTask(myTask)
    ```

  - 添加到事件队列

    ```dart
    Future(myTask)
    ```

- 延时任务

  ```dart
  Future.delayed(Duration(seconds:1),myTask)
  ```

- Future详解

  - Future是对未来结果的一个代理，封装了该任务的执行状态
    
  - Future只是创建一个事件到事件队列中
    
  - sync是同步方法，会立即执行

    ```
    Future.sync
    ```

    sync执行了其传入的函数后，立即创建任务到微任务队列中

  - then是注册回调

    ```
    Future(myTask).then(thenTask)
    ```

    - 执行完myTask，立即回调执行thenTask
    - 并不是创建新的Event事件到事件队列中,而是和Future共用一个事件循环
    - Future在then之前执行完，会创建一个task添加到微任务队列中

    ```dart
    //Future的函数体为null，意味着它不需要也没有事件循环，因此后续then也无法与它共享事件循环
    Future(() => null).then((_) => print('then 4'));
    ```

  - catchError处理异常

    ```
    Future(myTask).then(thenTask).catchError
    ```

    还可以通过then的参数onError来捕获异常

  - whenComplete

    ```
    Future(myTask).then(thenTask,onError:errortask).whenComplete(lastTask)
    ```

    异步任务无论成功与否都执行

  - wait等待多个任务全部完成后回调

    ```
    Future.wait([task1,task2,task3]).then(thenTask)
    ```

## 异步函数

- async和await

  ```
  test() async{
    var r=await doTask();
    print(r)
  }
  ```

  async只是语法糖，简化Future API的使用

  async标记的方法，只能由await来调用

- await不是阻塞等待，而是异步等待

  - 如果想同步等待，需要在调用异步函数时加上await，在所在方法上加上async

- Stream

  ```dart
  Stream.fromFutures([
    //2秒后返回结果
    Future.delayed(new Duration(seconds: 2), () {
      return "Android";
    }),
  
    //3秒后抛出一个异常
    Future.delayed(new Duration(seconds: 3), () {
      return AssertionError("error");
    }),
  
    //4秒后返回结果
    Future.delayed(new Duration(seconds: 4), () {
      return "Flutter";
    })
  ]).listen((result) {
    //打印接收的结果
    print(result);
  }, onError: (e) {
    //错误回调
    print(e.message);
  }, onDone: () {});
  ```

  Stream用来接收多个异步操作的结果

## Isolate

- Isolate

  - 非常耗时的任务添加到事件队列后，仍然可能导致阻塞

  - 协程

    - 不能共享内存
    - 不存在锁竞争
    - 只能通过发送消息通信

  - spawn

    主Isolate

    ```dart
    create_isolate async{
       ReceivePort rp=ReceivePort();
       SendPort port1=rp.sendPort;
       Isolate newIsolate=await Isolate.spawn(doWork,port1);
       rp.listen((message){})
    }
    ```

    新Isolate

    ```
    void doWork(SendPort prot1){
       ReceivePort rp=ReceivePort();
       SendPort port2=rp.sendPort;
       rp.listen((message){});
       prot1.send(port2);
    }
    ```

  - spawnUri

    主Isolate

    ```dart
    ReceivePort rp=ReceivePort();
    SendPort port1=rp.sendPort;
    Isolate newIsolate=await Isolate.spawnUri(Uri("path"),[参数列表],prot1)
    rp.listen((message){})
    //newIsolate.kill(priority: Isolate.immediate)
    ```

    新Isolate

    ```
    void main(args,SendPort port1){
        ReceivePort rp=ReceivePort();
        rp.listen((message){});
        port1.send([port2])
    }
    ```

- Flutter中创建Isolate

  ```dart
  create_new_isolate async{
    var result=await compute(doWork,"New Task")
  }
  void doWork(String value){
    return "$value"
  }
  ```

  - compute
    - 待执行的函数，不能是类的实例方法，可以是类的静态方法
    - 动态的消息类型，可以是被运行函数的参数

- 何时使用Future，何时使用Isolate
  - 根据平均执行时间来选择
    - 几百毫秒以上使用Isolate，低于则使用Future

# Dart特性

- 一切皆对象
  - 数值和操作符的父类都是num
  - 多行字符串拼接可以使用'''
  - 定义一个List、Map支持多种类型，可以List<num>
- 常量定义
  - const：表示变量在编译期间既能确定的值，在声明后就不能改
  - final：表示在运行时确定值，即在赋值后，就不能改
- 函数
  - 函数也是对象
  - 不支持重载，而是提供了可选命名参数

- 类

  - 没有访问修饰符

  - _不是类访问级别，而是库访问级别

  - 支持初始化列表

    ```dart
    class Point { 
      num x, y, z; 
      Point(this.x, this.y) : z = 0; // 初始化变量z 
      Point.bottom(num x) : this(x, 0); // 重定向构造函数 
      void printInfo() => print('($x,$y,$z)');}
    ```

  - 可以利用语法糖和初始化列表，简化构造函数

    ```dart
    ShoppingCart(this.name, this.code) : date = DateTime.now();
    ```

  - 命名构造函数

    ```dart
    //默认初始化方法，转发到withCode里 
    ShoppingCart({name}) : this.withCode(name:name, code:null); //withCode初始化方法，使用语法糖和初始化列表进行赋值，并调用父类初始化方法 
    ShoppingCart.withCode({name, this.code}) : date = DateTime.now(), super(name,0);
    ```

- 复用

  - 继承：子类复用父类的成员变量和方法实现，可以根据需要重写构造函数及父类
  - 接口：子类获取到的仅仅是接口的成员变量符号和方法符号，需要重新实现成员变量，以及方法的声明和初始化
  - 混入（Mixin）：with关键字

- 运算符

  - 判空

    - ?.`p.printInfo()`
    - ??=`a??=value`如果a为null，则赋值，否则跳过
    - ??`a??b`如果a不为null，取a，否则取b

  - 复写运算符

    ```dart
    class Vector { 
    num x, y; 
    Vector(this.x, this.y); // 自定义相加运算符，实现向量相加 
    Vector operator +(Vector v) => Vector(x + v.x, y + v.y); // 覆写相等运算符，判断向量相等 
    bool operator == (dynamic v) => x == v.x && y == v.y;}
    ```

    

    

