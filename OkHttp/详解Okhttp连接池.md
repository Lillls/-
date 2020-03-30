# 详解Okhttp连接池

## 为什么要用连接池

什么要设置连接池呢?

首先要知道Request Header中`keep-alive`属性

![浏览器发起请求header](http://xiaoyu-ipic.oss-cn-beijing.aliyuncs.com/2019-12-02-073430.jpg)

`keep-alive`就是客户端和服务端保持持久连接,使用同一个TCP连接来发送和接收多个HTTP请求/应答，而不是为每一个新的请求/应答打开新的连接的方法。

通常我们在发一个请求的时候会先打开一个Tcp连接---三次握手,然后再传输数据,最后关闭Tcp连接---四次挥手.

使用`keep-alive`会减少后续请求的延迟,较小的CPU和内存的使用.

## 怎么用?

OkHttp是通过`ConnectionPool`来实现和管理连接池的

如果不设置连接池的话,OkHttp默认实现了一个连接池

```java
connectionPool = new ConnectionPool();
```

```java
public ConnectionPool() {
  this(5, 5, TimeUnit.MINUTES);
}
```

当然我们也可以自己设置连接池

```java
OkHttpClient client = new OkHttpClient.Builder()
        .connectionPool(new ConnectionPool(maxIdleConnections,
                keepAliveDuration, timeUnit))
        .build();
```

这样就能实现连接池了

## 源码解析

先来看一下`ConnectionPool`这个类

```java
public final class ConnectionPool {
  final RealConnectionPool delegate;

  public ConnectionPool() {
    this(5, 5, TimeUnit.MINUTES);
  }

  public ConnectionPool(int maxIdleConnections, long keepAliveDuration, TimeUnit timeUnit) {
    this.delegate = new RealConnectionPool(maxIdleConnections, keepAliveDuration, timeUnit);
  }

  public int idleConnectionCount() {
    return delegate.idleConnectionCount();
  }

  public int connectionCount() {
    return delegate.connectionCount();
  }

  public void evictAll() {
    delegate.evictAll();
  }
}
```

可以看到ConnectionPool只是一个壳子,真正做事的`RealConnectionPool`

来看一下它的源码吧

### 成员变量

```java
//用于清除过期连接的线程.
private static final Executor executor = new ThreadPoolExecutor(0 /* corePoolSize */,
    Integer.MAX_VALUE /* maximumPoolSize */, 60L /* keepAliveTime */, TimeUnit.SECONDS,
    new SynchronousQueue<>(), Util.threadFactory("OkHttp ConnectionPool", true));
//最大空闲连接数
private final int maxIdleConnections;
//空闲连接最大存活时间
private final long keepAliveDurationNs;
//清理的runnable
private final Runnable cleanupRunnable = () -> {
  while (true) {
    long waitNanos = cleanup(System.nanoTime());
    if (waitNanos == -1) return;
    if (waitNanos > 0) {
      long waitMillis = waitNanos / 1000000L;
      waitNanos -= (waitMillis * 1000000L);
      synchronized (RealConnectionPool.this) {
        try {
          RealConnectionPool.this.wait(waitMillis, (int) waitNanos);
        } catch (InterruptedException ignored) {
        }
      }
    }
  }
};
//存放连接的队列
private final Deque<RealConnection> connections = new ArrayDeque<>();
final RouteDatabase routeDatabase = new RouteDatabase();
//一个标志,没有任何连接时,为false
boolean cleanupRunning;
```

既然是个池子,那重要的就是清理方法,添加方法和获取方法

### cleanup

先从真正清理的`cleanup`方法看起

```java
long cleanup(long now) {
  int inUseConnectionCount = 0;
  int idleConnectionCount = 0;
  RealConnection longestIdleConnection = null;
  long longestIdleDurationNs = Long.MIN_VALUE;

  synchronized (this) {
    for (Iterator<RealConnection> i = connections.iterator(); i.hasNext(); ) {
      RealConnection connection = i.next();
      //如果连接正在使用,inUseConnectionCount++,跳过本地循环
      if (pruneAndGetAllocationCount(connection, now) > 0) {
        inUseConnectionCount++;
        continue;
      }
			//空闲连接++,然后找到空闲时间最长的那个连接
      idleConnectionCount++;
      long idleDurationNs = now - connection.idleAtNanos;
      if (idleDurationNs > longestIdleDurationNs) {
        longestIdleDurationNs = idleDurationNs;
        longestIdleConnection = connection;
      }
    }
		
    //1.如果空闲时间>=keepAliveDurationNs或者空闲连接大于最大空闲连接数,直接清理
    //2.如果有空闲连接,那就返回keepAliveDurationNs - longestIdleDurationNs,也就是还能存活多少时间
    //3.如果只有使用的连接,那就返回最大空闲时间
    //4.都不符合那就说明没有连接,cleanupRunning置为false,返回-1,不需要进一步清理
    if (longestIdleDurationNs >= this.keepAliveDurationNs
        || idleConnectionCount > this.maxIdleConnections) {
      connections.remove(longestIdleConnection);
    } else if (idleConnectionCount > 0) {
      // A connection will be ready to evict soon.
      return keepAliveDurationNs - longestIdleDurationNs;
    } else if (inUseConnectionCount > 0) {
      return keepAliveDurationNs;
    } else {
      cleanupRunning = false;
      return -1;
    }
  }
  //关闭掉被清理的连接
  closeQuietly(longestIdleConnection.socket());

  //继续下次清理
  return 0;
}
```

然后在看`cleanupRunnable`做了什么

```java
private final Runnable cleanupRunnable = () -> {
  while (true) {
    long waitNanos = cleanup(System.nanoTime());
    if (waitNanos == -1) return;
    if (waitNanos > 0) {
      long waitMillis = waitNanos / 1000000L;
      waitNanos -= (waitMillis * 1000000L);
      synchronized (RealConnectionPool.this) {
        try {
          RealConnectionPool.this.wait(waitMillis, (int) waitNanos);
        } catch (InterruptedException ignored) {
        }
      }
    }
  }
};
```

应该不需要再详细解释了.看一张图来总结一下怎么进行清理的

![image-20191202170827740](http://xiaoyu-ipic.oss-cn-beijing.aliyuncs.com/2019-12-02-090828.png)

然后看一下判断是否为空闲连接的方法

### pruneAndGetAllocationCount

```java
private int pruneAndGetAllocationCount(RealConnection connection, long now) {
  List<Reference<Transmitter>> references = connection.transmitters;
  for (int i = 0; i < references.size(); ) {
    Reference<Transmitter> reference = references.get(i);

    if (reference.get() != null) {
      i++;
      continue;
    }

    // We've discovered a leaked transmitter. This is an application bug.
    TransmitterReference transmitterRef = (TransmitterReference) reference;
    String message = "A connection to " + connection.route().address().url()
        + " was leaked. Did you forget to close a response body?";
    Platform.get().logCloseableLeak(message, transmitterRef.callStackTrace);

    references.remove(i);
    connection.noNewExchanges = true;

    // If this was the last allocation, the connection is eligible for immediate eviction.
    if (references.isEmpty()) {
      connection.idleAtNanos = now - keepAliveDurationNs;
      return 0;
    }
  }

  return references.size();
}
```

代码很简单

1. 先拿到这个连接的transmitter的references ,transmitter就相当于一次请求,一次调用
2. 循环references
3. 把被GC回收掉从references中移除
4. 然后返回references size

这个应该算是利用一个引用计数的方法来判断,如果references.size>0,就说明还有请求正在使用这个连接,

清理到这就看完了,然后在看看是怎么往连接池里添加的.

### put

```java
void put(RealConnection connection) {
  assert (Thread.holdsLock(this));
  if (!cleanupRunning) {
    cleanupRunning = true;
    executor.execute(cleanupRunnable);
  }
  connections.add(connection);
}
```

`put`方法就很简单了,先判断清理线程是否运行,如果没运行,那就运行起来,然后向连接池里添加一个连接.