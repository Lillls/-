# 详解Flutter中的mixin

## 定义

在学习Dart语言的时候，看到过mixin这个关键字，它的定义是：

**在面向对象的语言中,mixins类是一个可以把自己的方法提供给其他类使用，但却不需要成为其他类的父类**。

先来看一下mixin这个关键字的特性

- 类似于用class定义一个类，用mixin定义一个 *拓展类*。
- 用mixin定义的类没有构造方法也就是不能被实例化,也就不能被继承，只能被实现
- 类似于 extend是继承的关键字,implements是实现的关键字。with是混入的关键字
- mixin可以混入多个

来看一下简单的用法

## 用法

首先定义一个mixin类

```dart
mixin MixinPrint {
  void print(String str) {
    print(str);
  }
}
```

然后定义一个类with它

```dart
class Computer with MixinPrint {
  void compute() {}
}
```

```
main() {
  Computer computer = new Computer();
  computer.compute();
  computer.print("有了打印的功能");
}
```

看起来很简单，但是在看flutter framework层的启动源码的时候，发现很复杂的用法。

## Flutter源码中的mixin

### 多mixin的调用栈问题

来看看Flutter源码中是怎么用的mixin

先定义一群mixin

```dart
abstract class BindingBase {

  BindingBase() {
    initInstances();
  }
  
  @protected
  @mustCallSuper
  void initInstances() {
  }
}

mixin GestureBinding on BindingBase implements HitTestable, HitTestDispatcher {
  @override
  void initInstances() {
    super.initInstances();
    //...do something..
  }
}

mixin ServicesBinding on BindingBase {
  @override
  void initInstances() {
    super.initInstances();
    //...do something..
  }
}

mixin SchedulerBinding on BindingBase, ServicesBinding {
  @override
  void initInstances() {
    super.initInstances();
    //...do something..
  }
}

mixin PaintingBinding on BindingBase, ServicesBinding {
  @override
  void initInstances() {
    super.initInstances();
    //...do something..
  }
}

mixin SemanticsBinding on BindingBase {
  @override
  void initInstances() {
    super.initInstances();
    //...do something..
  }
}

mixin RendererBinding on BindingBase, ServicesBinding, SchedulerBinding, GestureBinding, SemanticsBinding {
  @override
  void initInstances() {
    super.initInstances();
    //...do something..
  }
}

mixin WidgetsBinding on BindingBase, ServicesBinding, SchedulerBinding, GestureBinding, RendererBinding, SemanticsBinding {
  @override
  void initInstances() {
    super.initInstances();
    //...do something..
  }
}
```

使用的时候

```dart
class WidgetsFlutterBinding extends BindingBase with GestureBinding, ServicesBinding, SchedulerBinding, PaintingBinding, SemanticsBinding, RendererBinding, WidgetsBinding {
  
}
```

具体分析一下代码，

*WidgetsFlutterBinding*集成了*BindingBase*，BindingBase的构造方法调用了`initInstances()`,

所以在实例化*WidgetsFlutterBinding*的时候，会先调用*BindingBase*的构造方法,这个是毋容置疑的，但是会先调用*mixin*的`initInstances()`方法呢？而且这些*mixin*又互相with。

先看一下debug的层次图，在*BindingBase*的`initInstances()`的时候打了一个断点。

![](http://xiaoyu-ipic.oss-cn-beijing.aliyuncs.com/blog/2020-04-14-033642.png)

先说一下结论，

**最先调用的是最后一个mixin的`initInstances()`方法，然后`super.initInstances()`会按照*with*的顺序，从后向前依次调用。**

为什么会这样呢？

**当mixin之间存在继承关系的时候，**

通过调用栈可以看出来，从`new WidgetsFlutterBinding`到 `new BindingBase`,中间夹了很多自动生成的类，**也可以得出一个结论，先混入后继承**，每混入一次就多一层继承关系，

### 规律

- mixin是可以继承抽象类的，但不是用extends是用on关键字
- mixin之间也可以互相继承的，也是用on关键字来实现
- 当mixin类，同时on了其他的mixin，with这个mixin时，也需要把其他的mixin同时with
- 当with多个mixin并且有方法重复时，真正调用的是最后一个mixin的方法