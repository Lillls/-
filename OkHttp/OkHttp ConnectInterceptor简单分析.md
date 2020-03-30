# OkHttp ConnectInterceptor简单分析

先简单说一下几个关键类的基本用途

- RealConnection

  与远程服务器建立的连接,包括TCP,TLS握手

- Transmitter

  官方注释是 Bridge between OkHttp's application and network layers.

  用自己的话解释一下,我们使用一个call来发送请求,更底层的是connection的建立,request的写入和response的读取等,Transmitter就是call和更加底层东西的桥梁,把它们连接了起来

- Exchange

  主要作用是通过`ExchangeCodec`处理IO相关的操作,并且发送响应的event.

  例如把request写入流,把流读取成response等

- ExchangeCodec

  真正负责流的写入和读取

## ConnectInterceptor作用是什么?

先看一下这个类的注释

```java
/** Opens a connection to the target server and proceeds to the next interceptor. */
```

**打开与目标服务器的连接,然后交给下一个拦截器处理.**

## 怎么工作的?

废话不多说直接看源码吧

```java
public final class ConnectInterceptor implements Interceptor {
  public final OkHttpClient client;

  public ConnectInterceptor(OkHttpClient client) {
    this.client = client;
  }

  @Override public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Request request = realChain.request();
    Transmitter transmitter = realChain.transmitter();
    
    boolean doExtensiveHealthChecks = !request.method().equals("GET");
    Exchange exchange = transmitter.newExchange(chain, doExtensiveHealthChecks);

    return realChain.proceed(request, transmitter, exchange);
  }
}
```

1. 获取request

2. 获取transmitter.

3. 通过`transmitter.newExchange()`获取Exchange

4. 交给下个Interceptor处理,并且给`RealInterceptorChain`中的`exchange`赋值.

   这个地方跟其他的Interceptor不太一样,其他的Interceptor都是调用 `proceed(Request request)`,而ConnectInterceptor调用的是`proceed(Request request, Transmitter transmitter, @Nullable Exchange exchange)`,相当于个RealInterceptorChain中的exchange,参数赋值了.

最重要的就是这行代码了

```java
Exchange exchange = transmitter.newExchange(chain, doExtensiveHealthChecks);
```

```java
Exchange newExchange(Interceptor.Chain chain, boolean doExtensiveHealthChecks) {
	//... ...
  ExchangeCodec codec = exchangeFinder.find(client, chain, doExtensiveHealthChecks);
  Exchange result = new Exchange(this, call, eventListener, exchangeFinder, codec);

  synchronized (connectionPool) {
    this.exchange = result;
    this.exchangeRequestDone = false;
    this.exchangeResponseDone = false;
    return result;
  }
}
```

```java
public ExchangeCodec find(
      OkHttpClient client, Interceptor.Chain chain, boolean doExtensiveHealthChecks) {
    int connectTimeout = chain.connectTimeoutMillis();
    int readTimeout = chain.readTimeoutMillis();
    int writeTimeout = chain.writeTimeoutMillis();
    int pingIntervalMillis = client.pingIntervalMillis();
    boolean connectionRetryEnabled = client.retryOnConnectionFailure();

    try {
      RealConnection resultConnection = findHealthyConnection(connectTimeout, readTimeout,
          writeTimeout, pingIntervalMillis, connectionRetryEnabled, doExtensiveHealthChecks);
      return resultConnection.newCodec(client, chain);
    } catch (RouteException e) {
      trackFailure();
      throw e;
    } catch (IOException e) {
      trackFailure();
      throw new RouteException(e);
    }
  }
```

到这就不在往下深入了,findHealthyConnection的作用是获取一个connection,赋值给Transmitter.具体的源码以后再分析

然后在看看`newCodec()`方法

```java
ExchangeCodec newCodec(OkHttpClient client, Interceptor.Chain chain) throws SocketException {
  if (http2Connection != null) {
    return new Http2ExchangeCodec(client, this, chain, http2Connection);
  } else {
    socket.setSoTimeout(chain.readTimeoutMillis());
    source.timeout().timeout(chain.readTimeoutMillis(), MILLISECONDS);
    sink.timeout().timeout(chain.writeTimeoutMillis(), MILLISECONDS);
    return new Http1ExchangeCodec(client, this, source, sink);
  }
}
```

返回了一个ExchangeCodec实例,可能是Http1ExchangeCodec和Http2ExchangeCodec

## 总结

ConnectInterceptor做的事也很简单

1. 获取一个健康的connection赋值给Transmitter
2. 根据connection的是否是Http2Connection,来获取ExchangeCodec对象.
3. 获取一个Exchange对象
4. 给`RealInterceptorChain`的`exchange`赋值后,并交给下一个也就是`CallServerInterceptor`进行下一步处理.

