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

