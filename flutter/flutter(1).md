# flutter学习笔记 -- 示例代码分析

在Android studio（vs code）安装好插件后，可以创建我们的第一个Flutter程序。

``` Dart
import 'package:flutter/material.dart';

void main() => runApp(MyApp());
```
 运行MyApp这个对象

``` Dart
class MyApp extends StatelessWidget {
  // This widget is the root of your application.
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Code Sample for material.AppBar.actions',
      theme: ThemeData(
                primarySwatch: Colors.blue,
      ),
      home: MyStatelessWidget(),
    );
  }
}
```
 MyApp继承自无状态的StatelessWidget, 其中StatelessWidget继承自Widget, 对于Widget, flutter官网是这么说的:_Flutter Widget采用现代响应式框架构建，这是从 React 中获得的灵感，中心思想是用widget构建你的UI。 Widget描述了他们的视图在给定其当前配置和状态时应该看起来像什么。当widget的状态发生变化时，widget会重新构建UI，Flutter会对比前后变化的不同， 以确定底层渲染树从一个状态转换到下一个状态所需的最小更改（译者语：类似于React/Vue中虚拟DOM的diff算法）。_  

 MyApp这个对象重写了build这个方法，这个对象返回一个Widget,用于描述这个StatelessWidget

 StatelessWidget用于描述不依赖于其他数据状态的界面，对于有其他数据状态驱动的界面考虑使用StatefullWidget,例如时钟状态驱动或其他系统状态驱动。

build方法返回了一个title,这是用于描述App的，在Android，按下recent键会显示出来

theme, 主题，不再赘述。

home: 主界面，MyStatelessWidget()；

```
class MyStatelessWidget extends StatelessWidget {
  MyStatelessWidget({Key key}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('Hello World'),
        actions: <Widget>[
          IconButton(
            icon: Icon(Icons.shopping_cart),
            tooltip: 'Open shopping cart',
            onPressed: () {
              // ...
            },
          ),
        ],
      ),
    );
  }
}
```
MyStatelessWidget构造器中的key是父Widget中用于区别子Widget的标识符。

MyStatelessWidget的Buid返回了一个Scaffold, Scaffold是flutter中最基础的视觉布局组件，

AppBar标题栏

title标题栏文案

actions标题栏上标题后的一串Widget, 通常是若干IconButton

IconButton图标按钮

tooltip描述按下按钮的文案

icon、onPressed不再赘述

更多的View查看：https://flutterchina.club/widgets/
