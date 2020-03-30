# Part1  整体流程简单分析

## 简单使用

```java
OkHttpClient client = new OkHttpClient.Builder()
        .build();
Request request = new Request.Builder()
        .url("http://publicobject.com/helloworld.txt")
        .build();
client.newCall(request).enqueue(new Callback() {
    @Override
    public void onFailure(Call call, IOException e) {
        e.printStackTrace();
    }

    @Override
    public void onResponse(Call call, Response response) throws IOException {
        System.out.println(response.body().string());
    }
});
```

## 整体流程

这篇文章的标题是整体流程分析,只是让大家对整体的流程有个简单的认识,暂时就不分析每个类的作用是什么了.

直接通过图来简单看一下整体的流程是什么

![Untitled Diagram (http://xiaoyu-ipic.oss-cn-beijing.aliyuncs.com/2019-12-04-075222.png)](/Users/xiaoyu/Downloads/Untitled Diagram (4).png)

1. 通过`OkHttpClient.Builder().build()`创建出一个`OkHttpClient`对象
2. 通过`Request.Builder().buiuld()`创建一个`Request`对象
3. 调用`OkHttpClient.newCall(Request request)`方法,把上面的`Request`对象对象传进入,获取到一个`RealCall`对象,同时创建了一个`Transmitter`对象并复制给了`call`.
4. 然后调用`RealCall.enqueue(Callback responseCallback)`,传进入一个callback,就完成了一次异步请求.

这个流程很简单哈,最关键的是这个`RealCall.enqueue(Callback responseCallback)`方法.看看这个方法做了什么

## RealCall.enqueue

```java
@Override public void enqueue(Callback responseCallback) {
  synchronized (this) {
    if (executed) throw new IllegalStateException("Already Executed");
    executed = true;
  }
  transmitter.callStart();
  client.dispatcher().enqueue(new AsyncCall(responseCallback));
}
```

调用了`Dispatcher.enqueue(AsyncCall call)`,把responseCallback包装成了一个`AsyncCall`

### `AsyncCall`

```java
final class AsyncCall extends NamedRunnable {
}

public abstract class NamedRunnable implements Runnable {
  protected final String name;
  
  @Override public final void run() {
    String oldName = Thread.currentThread().getName();
    Thread.currentThread().setName(name);
    try {
      execute();
    } finally {
      Thread.currentThread().setName(oldName);
    }
  }

  protected abstract void execute();
}
```

`AsyncCall`就是一个实现了`Runnable`接口的类,它的父类在`run`里面调用了`execute`方法,所有`AsyncCall`重写`execute`,就相当于重写`run`了

然后再来看看`Dispatcher.enqueue(AsyncCall call)`

### `Dispatcher.enqueue(AsyncCall call)`

```java
void enqueue(AsyncCall call) {
  synchronized (this) {
    readyAsyncCalls.add(call);

    // Mutate the AsyncCall so that it shares the AtomicInteger of an existing running call to
    // the same host.
    if (!call.get().forWebSocket) {
      AsyncCall existingCall = findExistingCallWithHost(call.host());
      if (existingCall != null) call.reuseCallsPerHostFrom(existingCall);
    }
  }
  promoteAndExecute();
}
```

```java
private boolean promoteAndExecute() {
  assert (!Thread.holdsLock(this));

  List<AsyncCall> executableCalls = new ArrayList<>();
  boolean isRunning;
  synchronized (this) {
    for (Iterator<AsyncCall> i = readyAsyncCalls.iterator(); i.hasNext(); ) {
      AsyncCall asyncCall = i.next();

      if (runningAsyncCalls.size() >= maxRequests) break; // Max capacity.
      if (asyncCall.callsPerHost().get() >= maxRequestsPerHost) continue; // Host max capacity.

      i.remove();
      asyncCall.callsPerHost().incrementAndGet();
      executableCalls.add(asyncCall);
      runningAsyncCalls.add(asyncCall);
    }
    isRunning = runningCallsCount() > 0;
  }

  for (int i = 0, size = executableCalls.size(); i < size; i++) {
    AsyncCall asyncCall = executableCalls.get(i);
    asyncCall.executeOn(executorService());
  }

  return isRunning;
}
```

1. 先把AsyncCall放入准备请求队列readyAsyncCalls队列中
2. 循环readyAsyncCalls
3. 如果正在请求队列runningAsyncCalls小于最大请求数,并且当前请求host数小与最大host数,就放入正在请求队列和可请求队列中
4. 循环可请求队列,调用`asyncCall.executeOn()`
5. 然后就会调用`executorService.execute(this)`
6. 上面说过AsyncCall就是一个runnable了,然后就会调用到`AsyncCall.execute`
7. 然后调用`getResponseWithInterceptorChain`获取网络响应的结果
8. 回调给`callback.onResponse()`
9. 把`asyncCall`从相应的队列中移除

第7步,就是大家常说的Interceptor

## Interceptors

我们来看看都有哪些Interceptor.

```java
Response getResponseWithInterceptorChain() throws IOException {
  // Build a full stack of interceptors.
  List<Interceptor> interceptors = new ArrayList<>();
  interceptors.addAll(client.interceptors());
  interceptors.add(new RetryAndFollowUpInterceptor(client));
  interceptors.add(new BridgeInterceptor(client.cookieJar()));
  interceptors.add(new CacheInterceptor(client.internalCache()));
  interceptors.add(new ConnectInterceptor(client));
  if (!forWebSocket) {
    interceptors.addAll(client.networkInterceptors());
  }
  interceptors.add(new CallServerInterceptor(forWebSocket));

  Interceptor.Chain chain = new RealInterceptorChain(interceptors, transmitter, null, 0,
      originalRequest, this, client.connectTimeoutMillis(),
      client.readTimeoutMillis(), client.writeTimeoutMillis());

  boolean calledNoMoreExchanges = false;
  try {
    Response response = chain.proceed(originalRequest);
    if (transmitter.isCanceled()) {
      closeQuietly(response);
      throw new IOException("Canceled");
    }
    return response;
  } catch (IOException e) {
    calledNoMoreExchanges = true;
    throw transmitter.noMoreExchanges(e);
  } finally {
    if (!calledNoMoreExchanges) {
      transmitter.noMoreExchanges(null);
    }
  }
}
```

![image-20191204170928473](http://xiaoyu-ipic.oss-cn-beijing.aliyuncs.com/2019-12-04-090928.png)

从左到右依次处理Request,然后真正请求完网络之后再从右往左依次处理Response

这就是OkHttp整体的流程分析