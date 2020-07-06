# 基础组件

## Widget简介

- Widget
  - Widget和Element的关系
  - Widget的主要接口

- Context
  - 每一个Widget都对应一个Widget，context是当前widget在widget树中执行”相关操作“的一个句柄
- StatefulWidget
  - createState:每一个StatefulElement都对应一个State
- State
  - 作用
  - 属性
  - 生命周期
- 在Widget树中获取State对象
  - 通过context获取
    - findAncestorStateOfType()，沿着widget树向上查找指定类型的StatefulWidget对应的State对象
    - 一般都是通过ScaffoldState _state=Scaffold.of(context)
  - 通过GlobalKey
    - 这是一种在整个APP中引用element的机制
    - 设置Scaffold(key:_globalKey),然后在别处使用_globalKey.currentState获取该widget的State
      - globalKey.currentWidget\globalKey.currentElement
- 内置组件库
  - 基础组件
  - Material组件

# 状态管理

- Widget管理自身状态
  
  - 调用setState来设置状态
  
- 父Widget管理子Widget的状态

  - 属性传值

  - [InheritedWidget](https://time.geekbang.org/column/article/116382):数据从父widget到子widget逐层传递
  - Notification:从子widget向上传递至父widget，适用于子widget状态更新，发送通知上报的场景

  > 上面两种都需要依靠Widget树，就是只能在有父子关系的Widget之间进行数据共享

  - EventBus事件总线

- 混合状态管理

- 全局状态管理
  - 使用一个全局的事件总线
  - 使用一些状态管理的包，比如Provider、Mobx等

# 事件处理与通知

## 指针事件

- 指针事件表示用户交互的原始触摸数据

  - 手指接触屏幕PointerDownEvent
  - 手指在屏幕上移动PointerMoveEvent
  - 手指抬起PointerUpEvent

- 触摸事件发起时，Flutter会确定手指与屏幕发生接触的位置上，究竟有哪些组件，并将触摸事件交给最内层的组件去响应。

  - 事件从最内层的组件开始，沿着组件树向根节点向上冒泡分发
  - Flutter无法像浏览器冒泡那样取消或者停止事件进一步分发，只能通过hitTestBehavior 去调整组件在命中测试期内应该如何表现，比如把触摸事件交给子组件，或者交给其视图层级之下的组件去响应。

- FLutter提供了ListenerWidget，可以监听子Widget的原始指针事件

  ```dart
  Listener(
    child: Container(
      color: Colors.red,//背景色红色
      width: 300,
      height: 300,
    ),
    onPointerDown: (event) => print("down $event"),//手势按下回调
    onPointerMove:  (event) => print("move $event"),//手势移动回调
    onPointerUp:  (event) => print("up $event"),//手势抬起回调
  );
  ```

## 手势识别

- GestureDetector

  ```dart
  
  //红色container坐标
  double _top = 0.0;
  double _left = 0.0;
  Stack(//使用Stack组件去叠加视图，便于直接控制视图坐标
    children: <Widget>[
      Positioned(
        top: _top,
        left: _left,
        child: GestureDetector(//手势识别
          child: Container(color: Colors.red,width: 50,height: 50),//红色子视图
          onTap: ()=>print("Tap"),//点击回调
          onDoubleTap: ()=>print("Double Tap"),//双击回调
          onLongPress: ()=>print("Long Press"),//长按回调
          onPanUpdate: (e) {//拖动回调
            setState(() {
              //更新位置
              _left += e.delta.dx;
              _top += e.delta.dy;
            });
          },
        ),
      )
    ],
  );
  ```

- 手势竞技场

  - GestureDetector对每个手势都建立了一个GestureFactory，工厂类的内部使用手势识别类GestureRecognizer来确定当前处理的手势

## 事件总线EventBus

- 在第一个页面

  ```dart
  
  //建立公共的event bus
  EventBus eventBus = new EventBus();
  //第一个页面
  class _FirstScreenState extends  State<FirstScreen>  {
    String msg = "通知：";
    StreamSubscription subscription;
    @override
    initState() {
     //监听CustomEvent事件，刷新UI
      subscription = eventBus.on<CustomEvent>().listen((event) {
        setState(() {msg+= event.msg;});//更新msg
      });
      super.initState();
    }
    dispose() {
      subscription.cancel();//State销毁时，清理注册
      super.dispose();
    }
  }
  
  
  class SecondScreen extends StatelessWidget {
    @override
    Widget build(BuildContext context) {
      return new Scaffold(
        body: RaisedButton(
            child: Text('Fire Event'),
            // 触发CustomEvent事件
            onPressed: ()=> eventBus.fire(CustomEvent("hello"))
        ),
      );
    }
  }
  ```

## 通知Notification

- 每个节点都可以分发通知，通知会沿着当前节点向上传递，所有父节点都可以通过NotificationListener来监听通知，这称为通知冒泡。

  - 类似于用户触摸事件冒泡，但通知冒泡是可以终止的

- 自定义通知

  - Notification有一个dispatch(context)方法

    ```dart
    class NotificationRouteState extends State<NotificationRoute> {
      String _msg="";
      @override
      Widget build(BuildContext context) {
        //监听通知  
        return NotificationListener<MyNotification>(
          onNotification: (notification) {
            setState(() {
              _msg+=notification.msg+"  ";
            });
           return true;
          },
          child: Center(
            child: Column(
              mainAxisSize: MainAxisSize.min,
              children: <Widget>[
    //          RaisedButton(
    //           onPressed: () => MyNotification("Hi").dispatch(context),
    //           child: Text("Send Notification"),
    //          ),  
                Builder(
                  builder: (context) {
                    return RaisedButton(
                      //按钮点击时分发通知  
                      onPressed: () => MyNotification("Hi").dispatch(context),
                      child: Text("Send Notification"),
                    );
                  },
                ),
                Text(_msg)
              ],
            ),
          ),
        );
      }
    }
    
    class MyNotification extends Notification {
      MyNotification(this.msg);
      final String msg;
    }
    ```

- 阻止冒泡

  在NotificationListener里的onNotification参数，返回true，就能阻止冒泡

- 通知冒泡原理

  dispatch中执行了context.visitAncestorElements(visitAncestor)方法，该方法会向上遍历父级元素

  - visitAncestor是一个遍历回调参数，遍历过程中对遍历的父级元素都会执行该回调，遍历的终止条件是，已经遍历到根Element或某个遍历回调返回false。
    - visitAncestor会判断每一个遍历到的父级Widget是否是NotificationListener，如果不是，则返回true继续遍历，如果是则调用该NotificationListener的_dispatch方法
  - _dispatch里会回调onNotification回调，然后根据回调的结果决定是否继续向上遍历

# 路由和导航

- Route 是页面的抽象，主要负责创建对应的界面，接收参数，响应 Navigator 打开和关闭；

- 而 Navigator 则会维护一个路由栈管理 Route，Route 打开即入栈，Route 关闭即出栈，还可以直接替换栈内的某一个 Route。

## 基本路由

```dart
//第一个页面
Navigator.push(context, MaterialPageRoute(builder: (context) => SecondPage()));
//第二个页面
Navigator.pop(context)
```

- 要导航到一个新页面，需要创建一个MaterialPageRoute，MaterialPageRoute定义了路由创建及切换过渡动画的相关配置，可以针对不同平台，实现与平台页面切换动画风格一致的路由切换动画。
- 返回上一个页面，则需要调用 Navigator.pop 方法从堆栈中删除这个页面。

## 命名路由

```dart
MaterialApp( 
  //注册路由 
  routes:{ "second_page":(context)=>SecondPage(), },
);
//使用名字打开页面
Navigator.pushNamed(context,"second_page");
```

## 页面参数

- 打开页面时传递参数

```dart
//打开页面时传递字符串参数
Navigator.of(context).pushNamed("second_page", arguments: "Hey");
//取出路由参数
String msg = ModalRoute.of(context).settings.arguments as String;
```

- 关闭页面时，传递参数告知上一个页面处理结果

```dart
//打开页面，并监听页面关闭时传递的参数
Navigator.pushNamed(context, "third_page",arguments: "Hey").then(
                              (msg)=>setState(()=>_msg=msg)), )
//页面关闭时传递参数 
onPressed: ()=> Navigator.pop(context,"Hi")
```

# 数据持久化

## 文件

- 临时目录
  - 是系统可以随时清楚的目录，通常用来存放一些不重要的临时缓存数据，在Android上对应着getCacheDir返回的值
- 文档目录
  - 文档目录是只有在删除应用程序时才会被清除的目录，通常用来存放应用产生的重要数据，在Android上对应AppData目录

```dart

//创建文件目录
Future<File> get _localFile async {
  final directory = await getApplicationDocumentsDirectory();
  final path = directory.path;
  return File('$path/content.txt');
}
//将字符串写入文件
Future<File> writeContent(String content) async {
  final file = await _localFile;
  return file.writeAsString(content);
}
//从文件读出字符串
Future<String> readContent() async {
  try {
    final file = await _localFile;
    String contents = await file.readAsString();
    return contents;
  } catch (e) {
    return "";
  }
}
```

- 二进制流读写
  - 支持图片、压缩包等文件的读写

## SharedPreferences

- 文件适合大量的、有序的数据持久化，如果只需要缓存少量的键值对信息，则可以使用SharedPreferences

  ```dart
  //读取SharedPreferences中key为counter的值
  Future<int>_loadCounter() async {
    SharedPreferences prefs = await SharedPreferences.getInstance();
    int  counter = (prefs.getInt('counter') ?? 0);
    return counter;
  }
  
  //递增写入SharedPreferences中key为counter的值
  Future<void>_incrementCounter() async {
    SharedPreferences prefs = await SharedPreferences.getInstance();
      int counter = (prefs.getInt('counter') ?? 0) + 1;
      prefs.setInt('counter', counter);
  }
  ```

## 数据库

- sp只适用于持久化少量数据的场景，并不能用来存储大量数据

- 如果需要存储大量数据，会选用sqlite数据库

  - 相比于文件和sp，数据库在读写上更快、更灵活

  ```dart
  //创建数据库，通过数据库初始化语句，创建一个用于存放Student对象的studengs表
  final Future<Database> database = openDatabase(
    join(await getDatabasesPath(), 'students_database.db'),
    onCreate: (db, version)=>db.execute("CREATE TABLE students(id TEXT PRIMARY KEY, name TEXT, score INTEGER)"),
    onUpgrade: (db, oldVersion, newVersion){
       //dosth for migration
    },
    version: 1,
  );
  ```

  ```dart
  //存储数据
  Future<void> insertStudent(Student std) async {
    final Database db = await database;
    await db.insert(
      'students',
      std.toJson(),
      //插入冲突策略，新的替换旧的
      conflictAlgorithm: ConflictAlgorithm.replace,
    );
  }
  //插入3个Student对象
  await insertStudent(student1);
  ```

  ```dart
  //读取数据
  Future<List<Student>> students() async {
    final Database db = await database;
    final List<Map<String, dynamic>> maps = await db.query('students');
    return List.generate(maps.length, (i)=>Student.fromJson(maps[i]));
  }
  
  //读取出数据库中插入的Student对象集合
  students().then((list)=>list.forEach((s)=>print(s.name)));
  //释放数据库资源
  final Database db = await database;
  db.close();
  ```

# 与Native通信

## 方法通道MethodChannel

## Flutter调native

- 这是方法调用的消息传递机制

  ```dart
  //声明MethodChannelconst 
  platform = MethodChannel('samples.chenhang/utils');
  //异步等待方法通道的调用结果 
  result = await platform.invokeMethod('openAppMarket');
  ```

- native的响应实现

  ```java
  
  protected void onCreate(Bundle savedInstanceState) {
    //创建与调用方标识符一样的方法通道
    new MethodChannel(getFlutterView(), "samples.chenhang/utils").setMethodCallHandler(
     //设置方法处理回调
      new MethodCallHandler() {
        //响应方法请求
        @Override
        public void onMethodCall(MethodCall call, Result result) {
          //判断方法名是否支持
          if(call.method.equals("openAppMarket")) {
            //打开应用市场
          }else {
            //方法名暂不支持 
            result.notImplemented();
          }
        }
      });
  }
  ```

- 类似于网络异步调用

  - 在Android和Dart分别用了三种数据类型，Android返回是Integer，Dart端接收又变成了int类型
    - 因为Flutter会使用StandardMessageCodec对通道中传输的信息进行类似JSON的二进制序列化。以标准化序列传输

## 平台视图

- 方法通道解决的是原生能力逻辑复用的问题，那么平台视图解决的就是原生视图复用的问题
  - 平台视图允许开发者在Flutter里面嵌入原生系统的视图，并加入到Flutter的渲染树中

- Flutter如何实现原生视图的接口调用

  ```dart
  class SampleView extends StatelessWidget {
    @override
    Widget build(BuildContext context) {
      //使用Android平台的AndroidView，传入唯一标识符sampleView
      if (defaultTargetPlatform == TargetPlatform.android) {
        return AndroidView(viewType: 'sampleView');
      } else {
        //使用iOS平台的UIKitView，传入唯一标识符sampleView
        return UiKitView(viewType: 'sampleView');
      }
    }
  }
  ```

- 如何在原生系统实现接口

  ```java
  
  //视图工厂类
  class SampleViewFactory extends PlatformViewFactory {
      private final BinaryMessenger messenger;
      //初始化方法
      public SampleViewFactory(BinaryMessenger msger) {
          super(StandardMessageCodec.INSTANCE);
          messenger = msger;
      }
      //创建原生视图封装类，完成关联
      @Override
      public PlatformView create(Context context, int id, Object obj) {
          return new SimpleViewControl(context, id, messenger);
      }
  }
  //原生视图封装类
  class SimpleViewControl implements PlatformView {
      private final View view;//缓存原生视图
      //初始化方法，提前创建好视图
      public SimpleViewControl(Context context, int id, BinaryMessenger messenger) {
          view = new View(context);
          view.setBackgroundColor(Color.rgb(255, 0, 0));
      }
      
      //返回原生视图
      @Override
      public View getView() {
          return view;
      }
      //原生视图销毁回调
      @Override
      public void dispose() {
      }
  }
  ```

  - 在Activity中Flutter侧调用与视图工厂绑定起来

  ```java
  protected void onCreate(Bundle savedInstanceState) {
    ...
    Registrar registrar =    registrarFor("samples.chenhang/native_views");//生成注册类
    SampleViewFactory playerViewFactory = new SampleViewFactory(registrar.messenger());//生成视图工厂
  
  registrar.platformViewRegistry().registerViewFactory("sampleView", playerViewFactory);//注册视图工厂
  }
  ```

- 如何在运行时，动态调整原生视图的样式

  - 与基于声明式的 Flutter Widget，每次变化只能以数据驱动其视图销毁重建不同，原生视图是基于命令式的，可以精确地控制视图展示样式。
  - 我们会用到原生视图的一个初始化属性，即 onPlatformViewCreated：原生视图会在其创建完成后，以回调的形式通知视图 id，因此我们可以在这个时候注册方法通道，让后续的视图修改请求通过这条通道传递给原生视图。

  ```dart
  
  //原生视图控制器
  class NativeViewController {
    MethodChannel _channel;
    //原生视图完成创建后，通过id生成唯一方法通道
    onCreate(int id) {
      _channel = MethodChannel('samples.chenhang/native_views_$id');
    }
    //调用原生视图方法，改变背景颜色
    Future<void> changeBackgroundColor() async {
      return _channel.invokeMethod('changeBackgroundColor');
    }
  }
  
  //原生视图Flutter侧封装，继承自StatefulWidget
  class SampleView extends StatefulWidget {
    const SampleView({
      Key key,
      this.controller,
    }) : super(key: key);
  
    //持有视图控制器
    final NativeViewController controller;
    @override
    State<StatefulWidget> createState() => _SampleViewState();
  }
  
  class _SampleViewState extends State<SampleView> {
    //根据平台确定返回何种平台视图
    @override
    Widget build(BuildContext context) {
      if (defaultTargetPlatform == TargetPlatform.android) {
        return AndroidView(
          viewType: 'sampleView',
          //原生视图创建完成后，通过onPlatformViewCreated产生回调
          onPlatformViewCreated: _onPlatformViewCreated,
        );
      } else {
        return UiKitView(viewType: 'sampleView',
          //原生视图创建完成后，通过onPlatformViewCreated产生回调
          onPlatformViewCreated: _onPlatformViewCreated
        );
      }
    }
    //原生视图创建完成后，调用control的onCreate方法，传入view id
    _onPlatformViewCreated(int id) {
      if (widget.controller == null) {
        return;
      }
      widget.controller.onCreate(id);
    }
  }
  ```

- Android端接口

  ```java
  
  class SimpleViewControl implements PlatformView, MethodCallHandler {
      private final MethodChannel methodChannel;
      ...
      public SimpleViewControl(Context context, int id, BinaryMessenger messenger) {
          ...
          //用view id注册方法通道
          methodChannel = new MethodChannel(messenger, "samples.chenhang/native_views_" + id);
          //设置方法通道回调
          methodChannel.setMethodCallHandler(this);
      }
      //处理方法调用消息
      @Override
      public void onMethodCall(MethodCall methodCall, MethodChannel.Result result) {
          //如果方法名完全匹配
          if (methodCall.method.equals("changeBackgroundColor")) {
              //修改视图背景，返回成功
              view.setBackgroundColor(Color.rgb(0, 0, 255));
              result.success(0);
          }else {
              //调用方发起了一个不支持的API调用
              result.notImplemented();
          }
      }
    ...
  }
  ```

# 在原生应用中混编Flutter工程

- 混编方案
  - 将原生工程作为Flutter的子工程，由Flutter统一管理
  - 将Flutter作为原生工程共用的子模块，这是一种三端分离模式
    - 轻量级接入
    - 实现Flutter功能的热插拔
    - 将Flutter模块打包成aar和pod，这样原生工程就可以像引用其他第三方组件库那样快速接入Flutter

# Flutter的编译模式

## 编译模式

- Debug模式：对应Dart的JIT模式，打开所有的断言，所有的调试信息，服务扩展和调试辅助，支持热重载
- Release模式：对应Dart的AOT模式，关闭所有断言、调试辅助信息。优化了应用快速启动、代码快速执行、以及二进制包大小
- Profile模式：基本与Release模式一致，对了对服务扩展的支持

## Hot Reload

热重载是指在不终端app正常运行的情况下，动态注入修改后的代码片段。这离不开Flutter所提供的运行时编译能力

- JIT：指的是即时编译或运行时编译，在Debug模式中使用，可以动态下发和执行代码，启动速度快，但执行性能受运行时编译影响
- AOT：指的是提前编译或运行前编译，可以为特定的平台生成稳定的二进制代码，执行性能好、运行速度快

在Debug模式下，Flutter采用的是JIT动态编译，JIT 编译器将 Dart 代码编译成可以运行在 Dart VM 上的 Dart Kernel，而 Dart Kernel 是可以动态更新的。

![img](https://static001.geekbang.org/resource/image/2d/fa/2dfbedae7b95dd152a587070db4bb9fa.png)

热重载的流程可以分为扫描工程改动、增量编译、推送更新、代码合并、Widget 重建 5 个步骤：

1. 工程改动。热重载模块会逐一扫描工程中的文件，检查是否有新增、删除或者改动，直到找到在上次编译之后，发生变化的 Dart 代码。
2. 增量编译。热重载模块会将发生变化的 Dart 代码，通过编译转化为增量的 Dart Kernel 文件。
3. 推送更新。热重载模块将增量的 Dart Kernel 文件通过 HTTP 端口，发送给正在移动设备上运行的 Dart VM。
4. 代码合并。Dart VM 会将收到的增量 Dart Kernel 文件，与原有的 Dart Kernel 文件进行合并，然后重新加载新的 Dart Kernel 文件。
5. Widget 重建。在确认 Dart VM 资源加载成功后，Flutter 会将其 UI 线程重置，通知 Flutter Framework 重建 Widget。