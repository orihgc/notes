#### 基础组件

- Widget纯作为一个配置文件存在，可以理解为一个数据结构
- Element作为配置文件的实例化对象，具有生命周期的概念，承载构建上下文数据，且持有RenderObject,系统通过遍历Element来构建RenderObject数据
- 具体Layout，Paint交给RenderObject来完成

## Widget简介

- Widget
  1. Widget是来描述Element的配置数据
  2. Widget是一个不可变对象,所有变量都是final
3. WIdget可以复用，添加到Tree中不同位置
- 上下文
  1. Widget继承自一个诊断树DiagnosticableTree，debugFillProperties()复写父类的方法，设置诊断树的一些特性
  2. createElement：Flutter会调用此方法生成对应的Element，StatefulElement和StalessElement都会继承它
  3. canUpdate：newWidget与oldWidget的runtimeType和Key同时相等时就会用newWidget去更新oldWidget

### Context

- 表示当前widget在widget树中的上下文，每一个Widget都对应一个context，context是当前widget在widget树中执行”相关操作“的一个句柄
  - 提供了从当前widget向上遍历widget树以及按照widget类型查找父级widget的方法

### StatefulWidget

- 和StatelessWIdget一样，StatefulWIdget也是继承自Widget类，并重写createElement方法，不同的是，添加了一个新接口
  - createState:用于创建和StatefulWidget相关的状态

### State

- 作用
  1. 在WIdget构建时被同步读取
  2. 在Widget生命周期中可以被改变，可以调用setState方法通知Flutter引擎状态发生改变，Flutter引擎会重新调用build方法，重新构建Widget树
- 属性
  1. widget：表示与该State实例关联的widget实例，State只在第一次插入到树中时被创建，WIdget在重新构建时可能会变化，Flutter引擎会动态设置State.widget为新的Widget实例
  2. context
- 生命周期
  - initSate：Widget第一次插入到Widget树中时调用，只会调用一次
    - 状态初始化
    - 订阅子树的事件通知
  - didChangeDependencies：当State发生变化，对应Widget的子Widget的didChangeDependencies回调都会被调用
    - 系统语言Locale
    - 应用主题改变
  - build：构建WIdget子树
  - ressemble：为开发调试提供，在热重载时会调用
  - didUpdateWidget： 在重新构建Widget时，会调用Widget.canUpdate来检测Widget树同一位置的新旧节点，然后决定是否需要更新。当key和runtimeType都相同时返回true，则需要更新
  - deactivate：当State从树中移除时，会调用此回调
    - 一些场景下，flutter引擎会将State对象重新插入到树中，如果移除后没有重新插入到树中，则会调用dispose
  - dispose：当State被永久移除时调用

- 在子Widget树中获取父级StatefulWidget的State对象
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

## Element

- Element可以理解为Widget的实例，在Tree中有特定位置

- Widget是可被复用的，真正的Tree（即树状关系）其实是由Element来维护的,可以通过Element遍历Tree

- 生命周期

  创建

  - Flutter引擎调用Widget.craeteElement来创建Element，此时Element处于“initial”状态
  - 调用Element.mount将Element添加到Element树中，并将相关联的renderObjects添加到渲染树中，此时就显示在屏幕上了

  修改：有父WIdget的配置数据改变，State.build返回的Widget结构与之前不同

  - 为了复用旧的Element，Element会调用对应Widget的canUpdate方法，返回true，则复用旧Element，使用新Widget配置数据更新；反之，则参加一个新的Element

  移除

  - 1、祖先Element调用deactiveChild来移除子Element 
  - 2、相关的Element.renderObject也会从渲染树中移除
  - 3、调用element.deactive，此时Element状态变为inactive状态

  保留

  - inactive态的Element在当前动画最后一帧结束前都会保留，如果在动画执行结束后还未能重新变为active状态，则调用unmount移除它，此时element状态为defunct

  复用

  - 如果element或其祖先拥有一个Globalkey，将element从现有位置移除，调用active，再将其renderObject重新attch到渲染树

### updateChild

```dart
if (newWidget == null) {
  if (child != null)
    //deactiveChild并detachRenderObject
    //owner._inactiveElements.add(child);添加Element到BuildOwner维护的_inactiveElements列表
    deactivateChild(child);
  return null;
}
if (child != null) {
  if (child.widget == newWidget) {//若widget没变
    if (child.slot != newSlot)//renderObejct不一样
      updateSlotForChild(child, newSlot);//更新子element的renderObejct
    return child;
  }
  //若newWidget变了，通过canUpdate判断是否可以更新Widget
  if (Widget.canUpdate(child.widget, newWidget)) {
    if (child.slot != newSlot)
      updateSlotForChild(child, newSlot);
    child.update(newWidget);//可以则只更新widget
    assert(child.widget == newWidget);
    assert(() {
      child.owner._debugElementWasRebuilt(child);
      return true;
    }());
    return child;
  }
  deactivateChild(child);//否则要deactiveChild并inflateWidget完成Element的替换
  assert(child._parent == null);
}
```

### setState触发刷新

```dart
//setState方法
_element.markNeedsBuild();
//markNeedsBuild方法
_dirty = true;//将Element标记为Dirty
owner.scheduleBuildFor(this);
//scheduleBuildFor方法
_dirtyElements.add(element);//将element加入到_dirtyElements中记录
```

1. 在每一帧的绘制drawFrame中，会触发buildScope，其中触发了_dirtyElements的rebuild方法
2. rebuild会触发updateChild方法，触发child.update方法
3. update方法中触发rebuild，rebuild会进入updateChild

- 如此就进入了递归调用逻辑

- Flutter采用标记机制，一帧重新build一下

### unmount

```dart
//在super.drawFrame后触发finalizeTree
buildOwner.finalizeTree();
//finalizeTree
_inactiveElements._unmountAll();//在finalize方法中进行unmount操作
```

## RenderObject

- RenderObject和Element的关系
  - 并非所有Element都拥有与之相对应的RenderObject，只有类型是RenderObjectElement类型的才有。
    - ComponentElement，仅仅包裹_child，没有任何布局相关的，不需要参与测量绘制
    - Row、Column这种，则需要对应的RenderObject

### RenderObjectElement

RenderObjectElement使用RenderObjectWidget作为配置文件，在RenderTree中有一个与之对应的RenderObject用来执行具体的测量，绘制等操作

- 大多数RenderObject使用了三种常见的子模型
  - LeadRenderObjectElement
  - SingleChildRenderObjectElement
  - MultiChildRenderObjectElement
- 每个RenderObject节点都被分配了一个_slot的私有token，记录了自己这个子节点与父节点的其他所有child的关系。
  - 当子节点准备attach当前节点时，会将这个_slot回传给当前节点方便识别，并将子节点放到相应的位置
  - 当_slot发生变化时，Element的moveChildRenderObject将会被调用

#### insert

- MultiChildRenderObjectElement的mount方法：

```dart
  void mount(Element parent, dynamic newSlot) {
    //super.mout完成自身RenderObject的创建及attach
    super.mount(parent, newSlot);
    _children = List<Element>(widget.children.length);
    Element previousChild;
    for (int i = 0; i < _children.length; i += 1) {
      //通过inflateWidget，创建各个子Widget的Element，并调用其mout继续向下遍历
      final Element newChild = inflateWidget(widget.children[i], previousChild);
      _children[i] = newChild;
      previousChild = newChild;
    }
  }
```

- RenderObjectElement的mount方法

```dart
@override
void mount(Element parent, dynamic newSlot) {
  super.mount(parent, newSlot);
  //通过调用widget的createRenderObject完成创建，此处的widget类型为RenderObjectWidget，RenderObjectWidget为abstract类，createRenderObject由子类根据自己规则创建不同的RenderObject，如Stack中返回了RenderStack:
  _renderObject = widget.createRenderObject(this);
  _debugUpdateRenderObjectOwner();
  assert(_slot == newSlot);
  //attach
  attachRenderObject(newSlot);
  _dirty = false;
}
```

- RenderObjectElement的attachRenderObject方法

```dart
@override
void attachRenderObject(dynamic newSlot) {
  assert(_ancestorRenderObjectElement == null);
  _slot = newSlot;
  //遍历找到父节点
  _ancestorRenderObjectElement = _findAncestorRenderObjectElement();
  //完成向父节点的插入，insertChildRenderObject在RenderObjectElement中为抽象方法，需要子类实现。
  _ancestorRenderObjectElement?.insertChildRenderObject(renderObject, newSlot);
  final ParentDataElement<RenderObjectWidget> parentDataElement = _findAncestorParentDataElement();
  if (parentDataElement != null)
    _updateParentData(parentDataElement.widget);
}
```

#### update

- MultiChildRenderObjectElement的update

```dart
@override
void update(MultiChildRenderObjectWidget newWidget) {
  super.update(newWidget);
  assert(widget == newWidget);
  _children = updateChildren(_children, widget.children, forgottenChildren: _forgottenChildren);
  _forgottenChildren.clear();
}
```

首先调用super方法完成自身的更新，其次调用updateChildren()方法完成child的更新

- 更新自身：RenderObjectElement.update方法

```dart
@override
void update(covariant RenderObjectWidget newWidget) {
  super.update(newWidget);
  assert(widget == newWidget);
  assert(() {
    _debugUpdateRenderObjectOwner();
    return true;
  }());
  widget.updateRenderObject(this, renderObject);
  _dirty = false;
}
```

- 更新child：updateChildren

通过比较传入的List oldChildren和List newWidgets完成Element的移除，更新等操作，调用child的update方法，并返回新的子Element列表。

1. 先从顶部同时遍历oldChildren与newWidgets，比较元素是否匹配，如匹配，进行update并继续遍历，否则中断遍历
2. 如上述遍历未完成，从底部开始同时遍历oldChildren与newWidgets，比较元素是否匹配，匹配则继续遍历，否则中断遍历（此时并没有进行update，而是最后才统一进行这些元素的update，原因是为了保证child的update调用是正向有序的）---- **此时两列表只剩余中间部分**
3. 遍历oldChildren剩余部分，有key的记录进oldKeyedChildren，否则deactivate
4. 遍历newWidgets剩余部分，根据widget key向oldKeyedChildren查找，查找到，进行update并在oldKeyedChildren中移除，否则直接update
5. 再次从底部开始遍历，update步骤2中所述元素
6. 将oldKeyedChildren中剩余元素deactivate

#### detch

RenderObject的detach调用，可以参考attach流程，从Element的unmount方法进行分析

### RenderObject

- RenderObject作为基类，类中没有直接定义子model，也没有定义具体的坐标系或者布局协议
- 大多数情况下，我们自定义布局时并不直接继承RenderObject（复杂度太高），而是继承自RenderBox，RenderBox使用的是笛卡尔坐标系，如果想使用其他坐标系，可以直接继承自RenderObject
- RenderObject的布局应该仅取决于child的layout，并且只有在[layout]调用中将`parentUsesSize`设置为true时才应该如此。 此外，如果将其设置为true，则父项必须在要渲染子项时调用子项的[layout]，否则在子项更改其布局输出时将不会通知父项。
- RenderObject任何可能影响布局的变动，都应该调用markNeedsLayout

#### insert

insert方法位于ContainerRenderObjectMixin中,ContainerRenderObjectMixin作为RenderObject的mixin为当前RenderObject的各个child维护一个双向链表的关系，方便访问

- insert方法中，调用RenderObject的adoptChild，另外调用_insertIntoChildList维护双向链表guanxi

```dart
@override
void adoptChild(RenderObject child) {
  assert(_debugCanPerformMutations);
  assert(child != null);
  setupParentData(child);
  markNeedsLayout();
  markNeedsCompositingBitsUpdate();
  markNeedsSemanticsUpdate();
  super.adoptChild(child);
}
```

- markNeedsLayout

markNeedsLayout中并不是立即触发刷新，而是将布局信息_needsLayout置为true，即将当前布局标记为dirty状态，然后调度PipelineOwner刷新布局信息。这种机制可以批处理布局工作，合并多个顺序的写入，避免冗余的计算。

```dart
void markNeedsLayout() {
  assert(_debugCanPerformMutations);
  if (_needsLayout) {
    assert(_debugSubtreeRelayoutRootAlreadyMarkedNeedsLayout());
    return;
  }
  assert(_relayoutBoundary != null);
  //如果RenderObject的父布局计算布局信息时标识使用子布局的size,即_relayoutBoundary != this,那markNeedsLayout则将父布局标记为dirty状态，这是父布局和当前布局均需要更新，则由父布局调度PipelineOwner，那子布局自然也就跟着刷新了:
  if (_relayoutBoundary != this) {
    markParentNeedsLayout();
  } else {
    _needsLayout = true;
    if (owner != null) {
      assert(() {
        if (debugPrintMarkNeedsLayoutStacks)
          debugPrintStack(label: 'markNeedsLayout() called for $this');
        return true;
      }());
      //将当前RenderObject注册进PipelineOwner的待刷新列表中，然后触发“VisualUpdate”
      owner._nodesNeedingLayout.add(this);
      //刷新由当前布局处理
      owner.requestVisualUpdate();
    }
  }
}
```

```dart
@protected
void markParentNeedsLayout() {
  _needsLayout = true;
  final RenderObject parent = this.parent;
  if (!_doingThisLayoutWithCallback) {
    //遍历调用markNeedsLayout
    parent.markNeedsLayout();
  } else {
    assert(parent._debugDoingThisLayout);
  }
  assert(parent == this.parent);
}
```

- requestVisualUpdate

```dart
void requestVisualUpdate() {
  if (onNeedVisualUpdate != null)
    onNeedVisualUpdate();
}
```

onNeedVisualUpdate为PipelineOwner构造时外部传入，PipelineOwner的构造在RendererBinding中完成。

- onNeedVisualUpdate为PipelineOwner构造时外部传入，PipelineOwner的构造在RendererBinding中完成。

```dart
mixin RendererBinding on BindingBase, ServicesBinding, SchedulerBinding, GestureBinding, SemanticsBinding, HitTestable {
  @override
  void initInstances() {
    super.initInstances();
    _instance = this;
    _pipelineOwner = PipelineOwner(
      onNeedVisualUpdate: ensureVisualUpdate,
      onSemanticsOwnerCreated: _handleSemanticsOwnerCreated,
      onSemanticsOwnerDisposed: _handleSemanticsOwnerDisposed,
    );
```

RenderBinding为renderTree与FlutterEngine中间的胶水层，刚才调用PipelineOwner触发的刷新由RenderBinding传递给FlutterEngine,即通过ensureVisualUpdate。

Engine会再转回到Dart层window中的onBeginFrame与onDrawFrame。window中的onBeginFrame与onDrawFrame赋值由SchedulerBinding完成

### Key

- 每一个Widget构造时，都有一个可选参数Key
  1. Key是Widget、Element、SemanticNode的标识符
  2. 仅当新Widget的Key和该Element相关联的当前Widget的key相等时，Element才被更新
  3. 在有相同parent的Element中，Key必须是唯一的
  4. Key的子类是LocalKey或者GlobalKey
  5. 作为Widget构造时的可选参数，Key最终被传到顶层的Widget中
- Key的使用
  1. 控制一个Widget如何替换Tree中的另一个Widget的
  2. UI刷新时有两种方式，如果两个Widget的runtimeType与key两个属性均相等时(==)，则通过更新Element完成新Widget对旧Widget的替换(by calling [Element.update]),否则，旧Element会被从Tree中移除，通过新的Widget inflate出来新的Element并插入Tree中。
  3. 当Widget的Key是GlobalKey时，Element可以在Tree中任意移动（即改变Parent），而不丢失状态
  4. 通常，一个Widget只有一个child的时候，这个child是不需要Key的
- Key的作用
  1. mount方法中，如果Key是GlobalKey，调用key._register( **this** );方法，将Key，Element以键值对的形式保存在_registry中
  2. updateChild方法中更新child element
     - key相等，则直接更新Element，否则，需要先将旧的Element移除，再通过inflateWidget构造新的Element，并add到Tree中。这里被移除的Element，都通过deactiveChild加入到了_inactiveElements列表中
     - inflatedWidget中，如果newWidget的key是GlobalKey，则调用_retakeInactiveElement获取Key中记录的Element，而非新建，完成Element的复用。**这也就是GlobalKey的作用，可以使Element在刷新过程中被复用，不丢失状态，同时由于这个复用时不受Widget在Tree中位置限制的，也就可以时Element更改其在Tree中位置。** 
  3. unmount方法中，

# 状态管理

## 管理自身状态

- 调用setState来设置状态

## 父Widget管理子Widget的状态

- 属性传值

- [InheritedWidget](https://time.geekbang.org/column/article/116382):数据从父widget到子widget逐层传递
- Notification:从子widget向上传递至父widget，适用于子widget状态更新，发送通知上报的场景

> 上面两种都需要依靠Widget树，就是只能在有父子关系的Widget之间进行数据共享

- EventBus事件总线

## 全局状态管理

- 使用一个全局的事件总线，将主题状态改变对应为一个事件，在initState方法中订阅主题改变的事件。当主题状态改变时，订阅了该事件的组件就会收到通知，收到通知后setState方法重写build自己
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