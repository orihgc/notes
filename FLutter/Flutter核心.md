# Widget简介

- 各个树之间的关系
  - Widget纯作为一个配置文件存在，可以理解为一个数据结构
  - Element作为配置文件的实例化对象，具有生命周期的概念，承载构建上下文数据，且持有RenderObject,系统通过遍历Element来构建RenderObject数据
  - 具体Layout，Paint交给RenderObject来完成

- Widget
  1. Widget是来描述Element的配置数据
  2. Widget是一个不可变对象,所有变量都是final
  3. WIdget可以复用，添加到Tree中不同位置
- 上下文
  1. Widget继承自一个诊断树DiagnosticableTree，debugFillProperties()复写父类的方法，设置诊断树的一些特性
  2. createElement：Flutter会调用此方法生成对应的Element，StatefulElement和StalessElement都会继承它
  3. canUpdate：newWidget与oldWidget的runtimeType和Key同时相等时就会用newWidget去更新oldWidget

## Context

- 表示当前widget在widget树中的上下文，每一个Widget都对应一个context，context是当前widget在widget树中执行”相关操作“的一个句柄
  - 提供了从当前widget向上遍历widget树以及按照widget类型查找父级widget的方法

## StatefulWidget

- 和StatelessWIdget一样，StatefulWIdget也是继承自Widget类，并重写createElement方法，不同的是，添加了一个新接口
  - createState:用于创建和StatefulWidget相关的状态

## State

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

# Element

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

## updateChild

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

## setState触发刷新

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

## unmount

```dart
//在super.drawFrame后触发finalizeTree
buildOwner.finalizeTree();
//finalizeTree
_inactiveElements._unmountAll();//在finalize方法中进行unmount操作
```

# RenderObject

- RenderObject和Element的关系
  - 并非所有Element都拥有与之相对应的RenderObject，只有类型是RenderObjectElement类型的才有。
    - ComponentElement，仅仅包裹_child，没有任何布局相关的，不需要参与测量绘制
    - Row、Column这种，则需要对应的RenderObject

## RenderBox

Renderobject主要作用就是布局和绘制，它拥有一个parent和一个parentData。parentData是一个预留变量（插槽），它正是由parent来赋值的

RenderBox继承自RenderObject类，大多数情况下，直接使用RenderBox就可以了，除非遇到要自定义布局模型或坐标系统的情况

### 布局过程

- Constraints

  RenderBox中有一个size属性来保存控件的宽高，通过在组件树中从上往下传递BoxConstraints对象实现布局，这个对象可以限制子节点的最大和最小宽高，子节点必须遵守父节点的限制条件

  `layout(Constraints constraints, { bool parentUsesSize = false })`

  constraints是父节点对子节点大小的限制，parentUsesSize用于确定relayoutBoundary，表示子节点布局变化是否影响父节点，如果为true，当子节点布局发生变化时，父节点都会标记为需要重新布局

- relayoutBoundary

  - 我们通过markNeedsBuild()来标记Element为dirty，当一个Element标记为dirty时会重新build
  - 在RenderObject中也有一个类似的markNeedsBuild方法，判断自己是不是relayoutBoundary，如果不是就继续向parent查找，一直向上查找到是relayoutBoundary的RenderObject为止。

- performResize和performLayout

  RenderBox的测量和布局逻辑，RenderBox子类需要实现这两个方法来定制自身的布局逻辑

  - sizedByParent是该节点的大小是否仅通过parent传给它的constraints来确定，即该节点的大小与它自身的属性和其子节点无关。
    - 比如一个控件要永远充满parent的大小，那么sizedByParent就应该返回true，此时其子节点在performSize中就确定了，在后面的performLayout方法中将不会再被修改，此时performLayout只负责布局子节点
  - performLayout：除了完成自身布局，也必须完成子节点的布局
  - RenderBox的子类要定制布局算法不应该重写layout方法，因为对于任何RenderBox的子类，它的layout流程基本是相同的，不同之处只在具体的布局算法。具体的布局算法应该通过重写performSize()和performLayout两个方法来实现

- ParentData

  - layout结束，每个节点的位置就已经确定了，但节点的位置信息需要保存，而子节点在父节点的偏移数据正是通过RenderObject的parentData属性来保存的。
  - 在RenderBox中，其parentData属性默认是一个BoxParentData对象，该属性只能通过父节点的setupParentData()方法设置
  - 当然不只是存储偏移信息，所有和子节点特定的数据都可以存储到子节点的ParentData中
    - 如`ContainerBox`的`ParentData`就保存了指向兄弟节点的`previousSibling`和`nextSibling`，`Element.visitChildren()`方法也正是通过它们来实现对子节点的遍历。再比如`KeepAlive` 组件，它使用`KeepAliveParentDataMixin`（继承自`ParentData`） 来保存子节的`keepAlive`状态。

### 绘制过程

RenderObject可以通过paint()方法来完成具体绘制逻辑，流程和布局流程相似

```dart
void paint(PaintingContext context, Offset offset) { }
```

通过context.canvas可以取到Canvas对象，接下来就可以调用Canvas API来实现具体的绘制逻辑

如果节点有子节点，除了完成自身绘制逻辑之外，还要调用子节点的绘制方法

1. 首先判断有无溢出，没有则调用defaultPaint(context,offset)来完成绘制
2. 然后调用context.paintChild()来绘制子节点，并将layout阶段的offset加上自身偏移作为第二个参数传递给paintChild，如果子节点还有子节点，还会调用paint()方法，如此递归完成整个节点树的绘制
3. 当需要绘制的内容大小溢出当前空间时，将会执行paintOverflowIndicator来绘制溢出部分提示

- RepaintBoundary

  - 绘制边界需要由开发者RepaintBoundary组件自己指定

    RenderObject有一个isRepaintBoundary属性，该属性决定这个RenderObject重绘时独立于其父元素

  - 如果child.isRepaintBoundary,会调用_compositeChild()方法

    独立绘制是在不同的layer层上绘制的，正确使用`isRepaintBoundary`属性可以提高绘制效率，避免不必要的重绘。

  - 当调用 `markNeedsPaint()` 方法时，会从当前 `RenderObject` 开始一直向父节点查找，直到找到 一个`isRepaintBoundary` 为 `true`的`RenderObject` 时，才会触发重绘，这样便可以实现局部重绘。

## RenderObjectElement

RenderObjectElement使用RenderObjectWidget作为配置文件，在RenderTree中有一个与之对应的RenderObject用来执行具体的测量，绘制等操作

- 大多数RenderObject使用了三种常见的子模型
  - LeadRenderObjectElement
  - SingleChildRenderObjectElement
  - MultiChildRenderObjectElement
- 每个RenderObject节点都被分配了一个_slot的私有token，记录了自己这个子节点与父节点的其他所有child的关系。
  - 当子节点准备attach当前节点时，会将这个_slot回传给当前节点方便识别，并将子节点放到相应的位置
  - 当_slot发生变化时，Element的moveChildRenderObject将会被调用

### insert

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

### update

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

### detch

RenderObject的detach调用，可以参考attach流程，从Element的unmount方法进行分析

## RenderObject

- RenderObject作为基类，类中没有直接定义子model，也没有定义具体的坐标系或者布局协议
- 大多数情况下，我们自定义布局时并不直接继承RenderObject（复杂度太高），而是继承自RenderBox，RenderBox使用的是笛卡尔坐标系，如果想使用其他坐标系，可以直接继承自RenderObject
- RenderObject的布局应该仅取决于child的layout，并且只有在[layout]调用中将`parentUsesSize`设置为true时才应该如此。 此外，如果将其设置为true，则父项必须在要渲染子项时调用子项的[layout]，否则在子项更改其布局输出时将不会通知父项。
- RenderObject任何可能影响布局的变动，都应该调用markNeedsLayout

### insert

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

## Key

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
  3. unmount方法中
     - 如果Key是GlobalKey，增加自己从_registry中移除，一个以GlobalKey作为Key的Element，在mount和unmount的生命周期中，都是被记录在__registry中，索引为GlobalKey，这也是通过GlobelKey可以获取到Element的原因
- 不同Key的区别
  - GlobalKey
    - GlobalKey的作用上面已经讲过了，可以在一帧内，更改Element在tree中的位置而不丢失状态。
    - 同时GlobalKey中也提供了接口获取与之相关联的Element，Widget，Context等的实例。
    - GlobalKey需要在整个APP中唯一，另外也不可在一个Tree中包含两个具有同样GlobalKey的Widget。
    - 基于以上特性,GlobalKey是比较重的，如果不是有必要的的需求，尽量不要使用GlobalKey，而是推荐使用LocalKey.
  - LocalKey
    - 对于LocalKey的定义为不是GlobalKey的可以，作用仅为更新Element的时，完成Element的复用，与GlobalKey的区别则为仅当Element位置不变是才能完成复用。
    - 这也就仅要求LocalKey在具有相同parent的Element之间唯一就行。

# Flutter启动

到此整个 runApp 方法就分析完了，回顾一下整个过程，总结来说就是根据传入的 Widget 生成对应的 ElementTree 和 RenderTree，之后开始进行首帧的布局和绘制。其中 Widget 用来描述页面的属性，这个对象是十分轻量级的且是不可变的，同一个 Widget 可以描述多个 Element 的配置，Element 同时持有了 Widget 和 RenderObject，它汇总了所有的属性信息，重绘时只将需要修改的部分通知到 RenderObject。对于普通开发者，只需要关注最上层的 Widget 就可以了，十分简单高效。

## runApp

```dart
void runApp(Widget app) {
  //WidgetsFlutterBinding继承了BindingBase，这个类是将Widget架构和Flutter底层引擎连接的桥梁
  //ensureInitialized() 负责初始化以及返回实例，该方法会进行大量初始化操作。
    WidgetsFlutterBinding.ensureInitialized()
        ..attachRootWidget(app)
        ..scheduleWarmUpFrame();//进行第一次绘制
}
```

## Widget到Element到RenderObject的流程

- attachRootWidget

  ```dart
  //负责将 Widget、Element、RenderObject 三者关联起来
  void attachRootWidget(Widget rootWidget) {
    //实际上就是将传入的 Widget 包装到 RenderObjectToWidgetAdapter，它继承自RenderObjectWidget，负责将 Widget、Element、RenderObject 三者关联起来，其中的 RenderObject 对应前面初始化操作中创建的 renderView
      _renderViewElement = new RenderObjectToWidgetAdapter<RenderBox>(
          container: renderView,
          debugShortDescription: '[root]',
          child: rootWidget
      ).attachToRenderTree(buildOwner, renderViewElement);
    //其中 renderView 和 _renderViewElement 为 WidgetsFlutterBinding 的成员，可以看出每个 app 只存在一个 renderViewElement 和 renderView，并且一一对应。
  }
  ```

- attachToRenderTree

  ```dart
  //该方法负责创建根 Element，即 RenderObjectToWidgetElement，并且将 Element 与 Widget 进行关联，即创建出 WidgetTree 对应的 ElementTree。
  RenderObjectToWidgetElement<T> attachToRenderTree(BuildOwner owner, [RenderObjectToWidgetElement<T> element]) {
      if (element == null) {
          owner.lockState(() {
              // 创建根Element，RenderObjectToWidgetElement
              element = createElement();
              assert(element != null);
              element.assignOwner(owner);
          });
          owner.buildScope(element, () {
              // 这里会根据WidgetTree构建ElementTree
              element.mount(null, null);
          });
      } else {
        //如果 Element 已经创建过了，则将根 Element 中关联的 Widget 设为新的，由此可以看出 Element 只会创建一次，后面会进行复用。
          element._newWidget = this;
          element.markNeedsBuild();
      }
      return element;
  }
  ```

- mount

  如果 Element 是首次创建，会调用 mount，该方法由父类到子类会做下面几件事：

  1. **Element:** 将该 Element 标记为 active 的，设置 parent 为 null，slot 为 null，depth 为 1，如果对应的 widget 的 key 为 GlobalKey，在这里进行注册，即将Key与Element进行关联，设置 inheritedWidgets，用于由上至下传递数据。
  2. **RenderObjectElement:** 创建对应的 RenderObject，并 attach 到对应的 slot 位置。
  3. **RootRenderObjectElement:** 没做什么事，只是 assert 一下 parent 和 slot 为 null。
  4. **RenderObjectToWidgetAdapter:** 调用 `_rebuild()` 方法创建 ElementTree。

  如果不是首次创建，这种情况一般是多次调用了 `runApp` 方法，则更新对应的跟 Widget，并调用 `markNeedsBuild()` 方法准备重建 ElementTree。

- Rebuild

  ```dart
  // 实际上是调用updateChild更新ElementTree
  _child = updateChild(_child, widget.child, _rootChildSlot);
  ```

- updateChild

  ```dart
  // child表示要更新的Element，newWidget表示对应Element的Widget，newSlot用来标识Element的所在位置，返回该位置对应的新Element
  @protected
  Element updateChild(Element child, Widget newWidget, dynamic newSlot) {
      assert(() {
          // Debug下保证一个GlobalKey只对应一个Widget
          if (newWidget != null && newWidget.key is GlobalKey) {
              final GlobalKey key = newWidget.key;
              key._debugReserveFor(this);
          }
          return true;
      }());
      if (newWidget == null) {
          // 如果newWidget为空，child非空表示需要移除旧Element
          if (child != null)
              deactivateChild(child);
          // 将此Element的位置设为null
          return null;
      }
      if (child != null) {
          // 都非空且是相同Widget，更新位置标识即可
          if (child.widget == newWidget) {
              if (child.slot != newSlot)
                  updateSlotForChild(child, newSlot);
              // 更新后返回原Element
              return child;
          }
          // 若不是相同Widget则判断是否有相同的类型和相同的Key，是的话则更新Widget信息到Element
          if (Widget.canUpdate(child.widget, newWidget)) {
              if (child.slot != newSlot)
                  updateSlotForChild(child, newSlot);
              child.update(newWidget);
              assert(child.widget == newWidget);
              assert(() {
                  child.owner._debugElementWasRebuilt(child);
                  return true;
              }());
              // 更新后返回原Element
              return child;
          }
          // 若不符合更新的要求，则抛弃掉原Element，抛弃掉的Element会被回收到`_inactiveElements`列表中，不会立即被销毁
          deactivateChild(child);
          assert(child._parent == null);
      }
      // 其他情况下需要创建新的Element
      return inflateWidget(newWidget, newSlot);
  }
  ```

- inflateWidget

  ```dart
  @protected
  Element inflateWidget(Widget newWidget, dynamic newSlot) {
      final Key key = newWidget.key;
      if (key is GlobalKey) {
          // 先使用key去被回收的列表中看看是否有可以复用的Element
          final Element newChild = _retakeInactiveElement(key, newWidget);
          if (newChild != null) {
              newChild._activateWithParent(this, newSlot);
              // 找到后就复用被回收的Element，并且更新它的Child
              final Element updatedChild = updateChild(newChild, newWidget, newSlot);
              return updatedChild;
          }
      }
      // 没有可以复用的Element了，只能创建新的
      final Element newChild = newWidget.createElement();
      // mount新的Element
      newChild.mount(this, newSlot);
      return newChild;
  }
  ```

## 开始渲染

回到runApp里，最后一行调用WidgetsFLutterBinding实例的scheduleWarmUpFrame进行第一次绘制

- 这次 draw 完成之前都不会接收各种 event（触摸事件等等）
- 该方法主要调用了 `handleBeginFrame()` 和 `handleDrawFrame()` 两个方法

先了解一下Frame和FrameCallbacks的概念

- Frame：即每一帧的绘制过程，engine 通过 VSync 信号不断地触发 Frame 的绘制，实际上就是调用 SchedulerBinding 类中的 `_handleBeginFrame()` 和 `_handleDrawFrame()` 这两个方法，这个过程中会完成动画、布局、绘制等工作。
  - handleBeginFrame：执行了 transientCallbacks。
  - handleDrawFrame：这里进行 persistentCallbacks 和 postFrameCallbacks 的回调
- FrameCallbacks：Frame 绘制期间，有三个 callbacks 列表会被调用，这三个列表是 SchedulerBinding 类中的成员，它们的调用顺序如下：
  - transientCallbacks，由 Ticker 触发和停止，一般用于动画的回调。
  - persistentCallbacks，永久 callback，一经添加无法移除，由 `WidgetsBinding.instance.addPersitentFrameCallback()` 注册，这个回调处理了布局与绘制工作。
  - postFrameCallbacks，只会调用一次，调用后会被系统移除，可由 `WidgetsBinding.instance.addPostFrameCallback()` 注册，该回调一般用于State的更新。

## 真正的渲染

系统只在 persistentCallbacks 注册了一个回调，实际上为 RenderBinding 类中的 `drawFrame()` 方法以及其子类 WidgetsBinding 类中的 `drawFrame()` 方法：

```dart
@protected
void drawFrame() {
    pipelineOwner.flushLayout();
    pipelineOwner.flushCompositingBits();
    pipelineOwner.flushPaint();
    renderView.compositeFrame(); // this sends the bits to the GPU
    pipelineOwner.flushSemantics(); // this also sends the semantics to the OS.
}
```

```dart
@override
void drawFrame() {
    try {
        if (renderViewElement != null)
          //该方法会将被标记为 dirty 的 Element 进行 rebuild()
            buildOwner.buildScope(renderViewElement);
        super.drawFrame();
      //调用 buildOwner.finalizeTree() 还记得之前回收被抛弃的 Element 的列表 _inactiveElements 吗？列表中的 Element 们在这里会被彻底清除掉。
        buildOwner.finalizeTree();
    } finally {
        ...
    }
}
```

- pipelineOwner.flushLayout();

  ```dart
  void flushLayout() {
      ...
      while (_nodesNeedingLayout.isNotEmpty) {
          final List<RenderObject> dirtyNodes = _nodesNeedingLayout;
          _nodesNeedingLayout = <RenderObject>[];
          for (RenderObject node in dirtyNodes..sort((RenderObject a, RenderObject b) => a.depth - b.depth)) {
              if (node._needsLayout && node.owner == this)
                  node._layoutWithoutResize();
          }
      }
      ...
  }
  ```

  当 RenderObject 的宽高等布局相关的属性被 set 时（通过更改 Widget 的属性），它会被添加到 `_nodesNeedingLayout` 列表中，以标记为需要重新进行 layout。

- flushCompositingBits()

  ```dart
    void flushCompositingBits() {
      ...
      _nodesNeedingCompositingBitsUpdate.sort((RenderObject a, RenderObject b) => a.depth - b.depth);
      for (RenderObject node in _nodesNeedingCompositingBitsUpdate) {
        if (node._needsCompositingBitsUpdate && node.owner == this)
          node._updateCompositingBits();
      }
      _nodesNeedingCompositingBitsUpdate.clear();
      ...
    }
  ```

  该方法用于判断 RenderObject 是否拥有自己的 layer，如果该状态变化了，就会将该 RenderObject 标记为需要进行重绘的，然后在下面 `flushPaint()` 方法中进行重绘。

- flushPaint()

  ```dart
  void flushPaint() {
      ...
      final List<RenderObject> dirtyNodes = _nodesNeedingPaint;
      _nodesNeedingPaint = <RenderObject>[];
      // Sort the dirty nodes in reverse order (deepest first).
      for (RenderObject node in dirtyNodes..sort((RenderObject a, RenderObject b) => b.depth - a.depth)) {
          if (node._needsPaint && node.owner == this) {
              if (node._layer.attached) {
                  PaintingContext.repaintCompositedChild(node);
              } else {
                  node._skippedPaintingOnLayer();
              }
          }
      }
      ...
  }
  ```

  该方法进行了绘制过程，可以看出它不是重绘了所有 RenderObject，而是只重绘了被标记为 dirty 的 RenderObject，这些 RenderObject 会调用底层的 skia 库进行绘制。

- compositeFrame()

  ```dart
  void compositeFrame() {
      ...
      final ui.SceneBuilder builder = new ui.SceneBuilder();
      layer.addToScene(builder, Offset.zero);
      final ui.Scene scene = builder.build();
      if (automaticSystemUiAdjustment)
          _updateSystemChrome();
      ui.window.render(scene);
      scene.dispose();
      ...
  }
  ```

  这个方法将画好的 layer 传给 engine，该方法调用结束之后，手机屏幕就会显示出内容了。

- flushSemantics()

  Semantics 用于将一些 Widget 的信息传给系统用于搜索、App 内容分析等场景，这与 Flutter 绘制流程关系不大，这里略过。

  

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



