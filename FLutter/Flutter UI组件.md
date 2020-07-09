# 基础组件

## 文本、字体

- TextSpan：文本片段的样式和内容
- DefaultTextStyle：设置子Text都会继承这个样式，除非指定inherit来设置

## 按钮

- 自定义按钮

```dart
const FlatButton({
  ...  
  @required this.onPressed, //按钮点击回调
  this.textColor, //按钮文字颜色
  this.disabledTextColor, //按钮禁用时的文字颜色
  this.color, //按钮背景颜色
  this.disabledColor,//按钮禁用时的背景颜色
  this.highlightColor, //按钮按下时的背景颜色
  this.splashColor, //点击时，水波动画中水波的颜色
  this.colorBrightness,//按钮主题，默认是浅色主题 
  this.padding, //按钮的填充
  this.shape, //外形
  @required this.child, //按钮的内容
})
```

## 图片和ICON

Image的数据源可以是asset、文件、内存和网络

- ImageProvider
  - AssetImage从asset获取图片
  - NetworkImage从网络加载图片
- ICON
  - Material Design的字体图标
  - Icons包含了所有Material Design的静态变量定义

# 动画

## 简介

- Animation

  - 这是一个在一段时间内生成一个区间之间值的类，输出的值可以是线性的、曲线的、一个步进函数或者其他曲线函数，这是由Curve决定。
  - 根据Animation对象的控制方式，动画可以正向运行，也可以反向运行，甚至中间切换方向。
  - 在动画的每一帧中可以通过Animation获取动画当前的状态值

  - 动画通知
    - addListener():给Animation添加帧监听器，每一帧都会被调用
    - addStatusListener():可以给Animation添加动画状态改变监听器；动画开始、结束、正向或反向时会调用状态改变的监听器

- Curve

  - Curve来描述动画过程，动画过程可以是匀速的、匀加速的或者先加速后减速的

  - 通过`CurvedAnimation`来指定动画的曲线

    ```dart
    //Curves.easeIn
    final CurvedAnimation curve =
        new CurvedAnimation(parent: controller, curve: Curves.easeIn);
    ```

    `CurvedAnimation`可以通过包装`AnimationController`和`Curve`生成一个新的动画对象 ，我们正是通过这种方式来将动画和动画执行的曲线关联起来的

- AnimationController用于控制动画

  包括动画的启动、停止、反向播放等

  ```dart
  final AnimationController controller = new AnimationController( 
   duration: const Duration(milliseconds: 2000), 
   lowerBound: 10.0,
   upperBound: 20.0,
   vsync: this
  );
  ```

  在动画开始执行后开始生成动画帧，屏幕每刷新一次就是一个动画帧，在动画的每一帧，会随着根据动画的曲线来生成当前的动画值（`Animation.value`），然后根据当前的动画值去构建UI，当所有动画帧依次触发时，动画值会依次改变，所以构建的UI也会依次变化，所以最终我们可以看到一个完成的动画。 另外在动画的每一帧，`Animation`对象会调用其帧监听器，等动画状态发生改变时（如动画结束）会调用状态改变监听器。

- Tricker

  - 当创建一个AnimationController时，需要传递一个vsync参数，其接收一个TickerProvider类型的对象
  - Flutter应用在启动时都会绑定一个`SchedulerBinding`，通过`SchedulerBinding`可以给每一次屏幕刷新添加回调，而`Ticker`就是通过`SchedulerBinding`来添加屏幕刷新回调，这样一来，每次屏幕刷新都会调用`TickerCallback`。

- Tween

  - 默认情况下，`AnimationController`对象值的范围是[0.0，1.0]。如果我们需要构建UI的动画值在不同的范围或不同的数据类型，则可以使用`Tween`来添加映射以生成不同的范围或数据类型的值。

    ```dart
    final Tween doubleTween = new Tween<double>(begin: -200.0, end: 0.0);
    ```

    `Tween`继承自`Animatable`，而不是继承自`Animation`，`Animatable`中主要定义动画值的映射规则。

  - Tween.animate

    要使用Tween对象，需要调用其`animate()`方法，然后传入一个控制器对象。例如，以下代码在500毫秒内生成从0到255的整数值。

    ```dart
    final AnimationController controller = new AnimationController(
        duration: const Duration(milliseconds: 500), vsync: this);
    Animation<int> alpha = new IntTween(begin: 0, end: 255).animate(controller);
    ```


# 自定义组件

## 自定义组件方法简介

- 组合其他Widget

  `Container`就是一个组合组件，它由`DecoratedBox`、`ConstrainedBox`、`Transform`、`Padding`、`Align`等组件组成。

- 自绘：通过Flutter中提供的`CustomPaint`和`Canvas`来实现UI自绘。

- 实现RenderObject

## 自绘

- CustomPaint 是用以承接自绘控件的容器，并不负责真正的绘制。画布是Canvas。画笔是Paint

  而画成什么样子，则由定义了绘制逻辑的CustomPainter来控制

  - 对于画笔，可以配置它的各种属性，比如颜色、样式、粗细等
  - 对于画布，可以画点drawPoint、画线drawLine、画路径drawPath、画矩形drawRect、画圆drawCircle、画圆弧drawArc

