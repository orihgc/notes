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
- 混合状态管理
- 全局状态管理
  - 使用一个全局的事件总线
  - 使用一些状态管理的包，比如Provider、Mobx等

## 数据共享

根widget通过InheritedWidget共享了一个数据，那么就可以在任意子widget中获取共享的数据

- didChangeDependencies
  - 这是State的一个回调，会在依赖发生变化时回调
  - 这个依赖就是指子widget是否使用了InheritedWidget的数据

## 跨组件状态共享


