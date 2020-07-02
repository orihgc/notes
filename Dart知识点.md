# 异步

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

    - 并不是创建新的Event事件到事件队列中
    - Future在then之前执行完，会创建一个task添加到微任务队列中

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

- async和await

  ```
  test() async{
    var r=await doTask();
    print(r)
  }
  ```

  async只是语法糖，简化Future API的使用

  async标记的方法，只能由await来调用

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

# 文件操作

