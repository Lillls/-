# Handler详解

> 文中源码基于API 28,且部分有删减

##  Handler有什么用? 为什么要用Handler?

Android应用程序运行时会创建一个主线程就是我们常说的UI线程,而更新UI的操作只能在主线程进行.但是实际开发中,一些网络请求和耗时操作都要到子线程中进行,获取到执行结果之后想去修改]UI是不行的,所以Android设计了Handler,通过消息的机制来实现线程之间的通讯.

### 为什么设计成只能在主线程更新UI?

因为如果支持多线程修改 View 的话，由此产生的线程同步和线程安全问题将是非常繁琐,耗性能的.为了简化		系统的设计,只能在主线程更新UI.

## 介绍一下与Handler相关几个类

- Handler       

  负责send message和handle message

- Looper

  负责从MessageQueue里拿message,然后交给handler处理消息

- MessageQueue

  消息队列,负责储存消息.

- Message

  有几个特殊的Field需要注意一下

  1. long when发送延迟消息用到
  2. Handler target目标handler,也就是那个handler来处理它
  3. Runnalbe callback`handler.post(Runnable runnable)` ,这个runnable就是callback.

- ThreadLocal

他们之间的对应关系是这样的

- 一个Handler持有一个Looper
- 一个Looper持有一个MessageQueue
- 一个MessageQueue中包含多个Message

## Handler是怎么发送消息的, 相关源码是什么?

先简单的介绍一下源码,稍后从源码入手.

1. 首先Looper不断循环MessageQueue,从MessageQueue获取消息

2. 这个过程中穿插着handler发送消息
3. 从MessageQueue获取消息之后,就开始处理消息.

这个就是Handler简单原理.下面就开始源码解析

### Lopper的源码解析

#### Lopper的创建

Android是基于Java来进行开发的,java程序的入口是main方法,Android项目并不是没有main方法,只是隐藏的比较深而已.Android的main方法在**ActivityThread**中, 首先来看一下main方法.

```java
public static void main(String[] args) {
    //.....
    Looper.prepareMainLooper();
  	//....
    Looper.loop();
  	//...
}
```

然后看一下`Looper.prepareMainLooper()方法

```java
public static void prepareMainLooper() {
    prepare(false);
    synchronized (Looper.class) {
        if (sMainLooper != null) {
            throw new IllegalStateException("The main Looper has already been prepared.");
        }
        sMainLooper = myLooper();
    }
}
```

再点进去看`prepare(false)`;

```java
private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
}
```

回过头来看`prepareMainLooper()`中调用的`mylooper()`方法.

```java
public static @Nullable Looper myLooper() {
    return sThreadLocal.get();
}
```

就是通过ThreadLocal获取刚刚保存的looper.

------

看到这之后,就大概明白了Looper的创建过程

1. 首先在`main`方法中调用`Looper.prepareMainLooper()`,
2. `prepareMainLooper()`中调用`prepare()`,创建一个Looper对象,并通过`sThreadLocal`来保存它.
3. 回过头来看`prepareMainLooper()`方法,`sThreadLocal`获取到刚才创建的Looper,并复制给`sMainLooper`

解释一下`ThreadLocal`

1. `sThreadLocal`是Looper里的一个静态`ThreadLocal`对象,`ThreadLocal `是线程局部变量，它保存的变量每个线程都有各自独立的副本互不冲突。关于`ThreadLocal`详细介绍,下篇文章会讲到,

`sMainLooper`就是主线程的`looper`,所以主线程的`looper`也叫`mainLooper`

#### Looper循环过程

创建好`MainLopper`之后,就开始`Looper.loop()`,看一下它的源码

```java
public static void loop() {
    final Looper me = myLooper();
    //... ...
    final MessageQueue queue = me.mQueue;
		//... ...
    for (;;) {
        Message msg = queue.next(); // might block
        if (msg == null) {
            // No message indicates that the message queue is quitting.
            return;
        }
      	//... ... 
        try {
            msg.target.dispatchMessage(msg);  //分发消息
            dispatchEnd = needEndTime ? SystemClock.uptimeMillis() : 0;
        } finally {
            if (traceTag != 0) {
                Trace.traceEnd(traceTag);
            }
        }

        msg.recycleUnchecked();
    }
}
```

代码简化过后看起来很清楚

1. 通过`ThreadLocal`拿到`Looper`
2. 通过`Looper`拿到`MessageQueue`
3. 通过一个死循环,不短的从`MessageQueue`里获取`message`
4. 通过`msg.target`拿到`handler`,然后`handler`来分发事件,具体是怎么分发的,下面讲到`Handler`的时候会说.
5. 通过`msg.recycleUnchecked()`做一些回收清除工作.

说到这应该对Handler机制整体的运作方式有个简单的了解.现在再来看一下Handler是怎么发送消息的?

### Handler源码解析

看来看看`Handler`的几个构造方法

```java
public Handler() {
    this(null, false);
}

public Handler(Callback callback) {
  this(callback, false);
}

public Handler(Looper looper) {
  this(looper, null, false);
}

public Handler(Looper looper, Callback callback) {
  this(looper, callback, false);
}

public Handler(boolean async) {
  this(null, async);
}
public Handler(Callback callback, boolean async) {
  if (FIND_POTENTIAL_LEAKS) {
    final Class<? extends Handler> klass = getClass();
    if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
        (klass.getModifiers() & Modifier.STATIC) == 0) {
      Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
            klass.getCanonicalName());
    }
  }

  mLooper = Looper.myLooper();
  if (mLooper == null) {
    throw new RuntimeException(
      "Can't create handler inside thread " + Thread.currentThread()
      + " that has not called Looper.prepare()");
  }
  mQueue = mLooper.mQueue;
  mCallback = callback;
  mAsynchronous = async;
}

public Handler(Looper looper, Callback callback, boolean async) {
  mLooper = looper;
  mQueue = looper.mQueue;
  mCallback = callback;
  mAsynchronous = async;
}
```

可以看到Handler的构造方法足足有7个, 其实别看它多,其实就是为它的成员变量赋值.

我们主要关注`mLooper`,`mQueue`,`mCallback`这几个成员变量就够了,下面结合简单的使用来分析这个成员变量的意思

#### Handler简单使用

```java
Handler handler = new Handler() {
  @Override
  public void handleMessage(Message msg) {
    super.handleMessage(msg);
  }
};

Message msg = Message.obtain();
handler.sendMessage(msg);
```

我们简单来看看实例化的时候都做了什么

```java
public Handler() {
    this(null, false);
}

public Handler(Callback callback, boolean async) {
  //... ...
  mLooper = Looper.myLooper();
  if (mLooper == null) {
    throw new RuntimeException(
      "Can't create handler inside thread " + Thread.currentThread()
      + " that has not called Looper.prepare()");
  }
  mQueue = mLooper.mQueue;
  mCallback = callback;
  mAsynchronous = async;
}
```

在我们不设置`Looper`的情况下,会调用`Looper.myLooper()`,通过`sThreadLocal`来获取到当前线程的`Looper`

因为我这段代码是在主线程运行的,所有会直接获取到`mainLooper`.

如果我在子线程,没有设置`looper`,`Looper.myLooper()`也获取不到的话就会抛出一个异常.**在子线程实例化`Handler`,必须指定`looper`**.

然后通过`looper`获取获取`MessageqQueue`.

#### Handler发送消息

```java
public final boolean sendMessage(Message msg)
public final boolean sendMessageDelayed(Message msg, long delayMillis)
public final boolean sendEmptyMessage(int what)
public final boolean sendEmptyMessageDelayed(int what, long delayMillis)
public boolean sendMessageAtTime(Message msg, long uptimeMillis)
  
public final boolean post(Runnable r)
public final boolean postDelayed(Runnable r, long delayMillis)
//...
```

其实别看这么多发消息的方法,实际上大概就分为两种

1. 一种就是`sendMessage(Message msg)`
2. 另一种就是`post(Runnable r)`

来看看这两个有什么区别 ,先看 `post(Runnable r)`做了什么

```java
public final boolean post(Runnable r){
   return  sendMessageDelayed(getPostMessage(r), 0);
}

private static Message getPostMessage(Runnable r) {
  Message m = Message.obtain();
  m.callback = r;
  return m;
}
```

原来其实`Runnable`也会被封装为`Message`,并用一个`callback`字段标记它,这个`callback`字段后面会用到.

上面一串发送消息的代码,最终都会调用到

```java
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    msg.target = this;
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```

1. 将当前的handler赋值给`msg.target`
2. `queue.enqueueMessage(msg, uptimeMillis)`,这段代码就是把msg插入到消息队列中,具体怎么插入的,下篇文章在解释.

这就是`Handler`发送消息的机制,下面来说一说`Handler`处理消息的机制.

#### Handler处理消息

上面再将`Looper.loop`的时候,通过`msg.target`拿到`handler`,通过`handler.dispatchMessage`来分发消息,先来看看`dispatchMessage`(),这个方法

```java
public void dispatchMessage(Message msg) {
  if (msg.callback != null) {
    handleCallback(msg);
  } else {
    if (mCallback != null) {
      if (mCallback.handleMessage(msg)) {
        return;
      }
    }
    handleMessage(msg);
  }
}
```

`msg.callback`上面说了,就是我们post过来的`Runnable`

这个`mCallback`,就是构造方法传进来callback

```java
public interface Callback {
    /**
     * @param msg A {@link android.os.Message Message} object
     * @return True if no further handling is desired
     */
    public boolean handleMessage(Message msg);
}
```

看到这分发流程就很清晰了

1.先判断是否为runnable,如果是就执行runnable,分发结束

2.然后是否设置了callback,如果设置了且返回值为true,分发结束

3.调用handleMessage来处理消息

![image-20191114202210776](/Users/xiaoyu/Library/Application Support/typora-user-images/image-20191114202210776.png)

OVER, 至于消息是怎么被插入到消息队列的,发送delay消息是什么原理和ThreadLocal原理,请听下回分解