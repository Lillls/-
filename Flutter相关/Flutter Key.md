# Flutter Key

## key是什么？

 The [key] property controls how one widget replaces another widget in the
 tree. If the [runtimeType] and [key] properties of the two widgets are
 [operator==], respectively, then the new widget replaces the old widget by
 updating the underlying element (i.e., by calling [Element.update] with the
 new widget). Otherwise, the old element is removed from the tree, the new
 widget is inflated into an element, and the new element is inserted into the
tree.

 「Key」属性控制一个widget如何替换widget tree上的其他小部件。

 如果两个小部件的「runtimeType」和「runtimeType」属性是

  「operator ==」，然后新的Widget将通过底层Element的更新替换旧的widget。

相反的，旧的Element将从Element Tree中移除，新的Widget将被填充到Element中，然后添加到Element中。

## key有什么作用：

1. Widget在Widget Tree移动时，保留其状态
2. 保留用户的滚动状态
3. 修改List时保留状态

## 什么时候需要key

如果小部件是StatelessWidget,不需要key.

先看一下widget的源码

```dart
@immutable
abstract class Widget extends DiagnosticableTree {
  const Widget({ this.key });
  final Key key;
  @protected
  Element createElement();
  @override
  String toStringShort() {
    return key == null ? '$runtimeType' : '$runtimeType-$key';
  }
  @override
  void debugFillProperties(DiagnosticPropertiesBuilder properties) {
    super.debugFillProperties(properties);
    properties.defaultDiagnosticsTreeStyle = DiagnosticsTreeStyle.dense;
  }
  static bool canUpdate(Widget oldWidget, Widget newWidget) {
    return oldWidget.runtimeType == newWidget.runtimeType
        && oldWidget.key == newWidget.key;
  }
}

```

首先`@immutable`表明widget是不可变的,它只相当于一份配置.虽然我们将颜色保存在了StatefulWidget中,但是StatefulWidget并不会保存这个,而是通过StatefulElement来持有这个状态.它是通过canUpdate来判断是否要更新. 在`canUpdate`,设置Key以后既要比较runtimeType,又要比较key.这时候key不一致,就会更新.

## 把key用在哪

## 用哪个key

## widget的装载过程

以StatelessWidget为例

```dart
abstract class StatelessWidget extends Widget {
  /// Initializes [key] for subclasses.
  const StatelessWidget({ Key key }) : super(key: key);
  /// Creates a [StatelessElement] to manage this widget's location in the tree.
  ///
  /// It is uncommon for subclasses to override this method.
  @override
  StatelessElement createElement() => StatelessElement(this);

  @protected
  Widget build(BuildContext context);
}
```

先明白一点，StatelessWidget在创建的时候会调用`createElement()`,创建一个`StatelessElement`,这个后续再说。然后看一下`StatelessElement`的代码

```dart
/// An [Element] that uses a [StatelessWidget] as its configuration.
class StatelessElement extends ComponentElement {
  /// Creates an element that uses the given widget as its configuration.
  StatelessElement(StatelessWidget widget) : super(widget);

  @override
  StatelessWidget get widget => super.widget;

  @override
  Widget build() => widget.build(this);

  @override
  void update(StatelessWidget newWidget) {
    super.update(newWidget);
    assert(widget == newWidget);
    _dirty = true;
    rebuild();
  }
}
```

通过它的构造函数，将`Widget`传给了`StatelessElement`，然后保留着widget的引用。然后调用`build`方法，实际上会调用`widget`的`build`方法。然后又会创建新的widget和element，分别挂载到对应的Widget Tree和Element Tree.

以下面的代码为例.

```dart
class WidgetTree extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Container(
      child: Padding(
        padding: const EdgeInsets.all(8.0),
        child: Text("WidgetTree Demo"),
      ),
    );
  }
}
```

![image-20191112200232729](http://xiaoyu-ipic.oss-cn-beijing.aliyuncs.com/blog/2019-11-12-120233.png)

会得出一个结论**`Element`和`widget`是一一对应关系.**

看到这可能会有一个问题,最开始的widget或者element是怎么被加载出来呢?

我们先看一个小例子

```dart
void main() => runApp(MyApp());

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Demo',
      theme: ThemeData(
        primarySwatch: Colors.blue,
      ),
      home: MyHomePage(),
    );
  }
}

class MyHomePage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: WidgetTreeSample(),
    );
  }
}

class WidgetTreeSample extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Container(
      child: Padding(
        padding: const EdgeInsets.all(8.0),
        child: Text("WidgetTree Demo"),
      ),
    );
  }
}
```

众所周知，程序的入口是main方法。在main方法中调用了` runApp(Widget app)`,我的App也是一个Widget!!!!

这样看起来就有点清楚了,在调用runApp的时候会加载最初始的widget和element.看一下代码究竟是不是这样.

来看看` runApp(Widget app)`这个方法的源码。

```dart
void runApp(Widget app) {
  WidgetsFlutterBinding.ensureInitialized()
    ..attachRootWidget(app)
    ..scheduleWarmUpFrame();
}
```

然后点进去`attachRootWidget(app)`

```dart
void attachRootWidget(Widget rootWidget) {
  _renderViewElement = RenderObjectToWidgetAdapter<RenderBox>(
    container: renderView,
    debugShortDescription: '[root]',
    child: rootWidget,
  ).attachToRenderTree(buildOwner, renderViewElement);
}
```

