#ThreadLocal源码简单分析

## ThreadLocal有什么用?

这个类提供**线程局部变量**,这些变量不同于其他常用变量的是:每个线程通过`set`和`get`方法,都有他们独立初始化的副本.

简单来说用ThreadLocal可以实现线程局部变量.能够实现这个变量的线程隔离.

## ThreadLocal怎么用

```java
val threadLocal = ThreadLocal<String>()
threadLocal.set("Hello")
thread {
    threadLocal.set("World")
    println("子线程===>" + threadLocal.get())
}
Thread.sleep(1000)
println("主线程===>" + threadLocal.get())
```

![image-20191118112621287](http://xiaoyu-ipic.oss-cn-beijing.aliyuncs.com/blog/2019-11-18-032621.png)

这就是ThreadLocal的简单使用,这个例子也验证了ThreadLocal的作用.变量的线程隔隔离:在子线程中对threadLocal的值进行更改,不会影响到主线程的threadLocal的值

## ThreadLocal怎么实现这个功能的?

先看一下ThreadLocal的`set`方法.

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
```

然后分别点进去`getMap(t)`和`createMap(t, value)`

```java
ThreadLocalMap getMap(Thread t) {
  return t.threadLocals;
}
```

```java
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

在来看看t.threadLocals是什么

```java
ThreadLocal.ThreadLocalMap threadLocals = null;
```

看到这`set`方法调用过程就应该很明白了

1. 先判断Thread.threadLocals是否为空
2. 如果不为空就直接调用ThreadLocalMap.set()
3. 如果为空就实例化一个ThreadLocalMap对象,并赋值给Thread.threadLocals字段.

然后再来看看`get`方法

```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
```

基本上和`set`方法差不多.或者调用ThreadLocalMap.get(),或者初始化ThreadLocalMap.

通过这两个方法,可以看出ThreadLocal主要功能是由ThreadLocalMap来实现的,来看看ThreadLocalMap的源码是是怎么实现的

### ThreadLocalMap



借用一张图来梳理这个执行流程

![img](http://xiaoyu-ipic.oss-cn-beijing.aliyuncs.com/2019-11-18-080416.jpg)

参考:

https://juejin.im/post/5c99c7c8f265da60e65ba56d

