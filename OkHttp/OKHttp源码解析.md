# OKHttp源码解析

1. 当请求数大于最大请求数后,什么时候回执行大于最大请求数的请求.
2. http 1.0 1.1 2.0版本变化
3. connectionSpec????
4. Max Request和Connection Pool有什么区别
5. 缓存模型
6. 连接池



OkHttpClient

1. dispatcher                     

   调度器.根据MaxRequest maxRequestsPerHost 选择把请求放入到runningAsyncCalls还是readyAsyncCalls

2. proxy

   设置代理

3. connectionSpec 

   连接规范,可以选择http,https,TLS版本之类.

InterceptorChain

## OkHttp的线程模型

从Dispatcher中的executorService方法可以看出:

```java
public synchronized ExecutorService executorService() {
  if (executorService == null) {
    executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
                                             new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp Dispatcher", false));
  }
  return executorService;
}
```

- 核心线程数为0
- 最大线程数不限制
- 非核心线程数空闲状态存活时间60s
- 任务阻塞队列为`SynchronousQueue`

线程池的相关介绍可以看https://blog.csdn.net/LilllS/article/details/103400154这篇文章

## OkHttp的缓存策略

### Http的缓存策略

大概的流程是:先查看本地是否有缓存,获取缓存规则,分以下两种

- 强制缓存

  - 如果缓存未失效,直接使用强制缓存,不访问网络
  - 如果缓存已失效,请求网络返回数据后,返回缓存数据和缓存规则,然后存储

- 对比缓存

  获取缓存数据的标识,向服务器验证缓存是否失效

  - 若未失效,使用本地缓存
  - 若以失效,返回最新数据和缓存规则,然后存储.

**基于对比缓存的流程下，不管是否使用缓存，都需要向服务器发送请求,在服务端验证缓存是否失效**

**强制缓存如果生效，不需要再和服务器发生交互,且强制缓存优先级高于对比缓存**

详细http的缓存策略请看这篇博客,写的很详细

https://www.cnblogs.com/chenqf/p/6386163.html

### 简单使用

先来看看怎么使用吧

```java
OkHttpClient client = new OkHttpClient.Builder()
                      .cache(new Cache(cacheDir, cacheSize))
                      .build();
```

很简单的代码就添加缓存的功能

还可以对单个Request用`CacheControl`进行cache控制

```java
Request request = new Request.Builder()
    .cacheControl(new CacheControl.Builder().noCache().build())
    .url("http://publicobject.com/helloworld.txt")
    .build();
```

先来看看`cache`是什么东西,代码有删减

```java
public final class Cache implements Closeable, Flushable {
  ... ...
  final InternalCache internalCache = new InternalCache() {
    @Override public Response get(Request request) throws IOException {
      return Cache.this.get(request);
    }

    @Override public CacheRequest put(Response response) throws IOException {
      return Cache.this.put(response);
    }

    @Override public void remove(Request request) throws IOException {
      Cache.this.remove(request);
    }

    @Override public void update(Response cached, Response network) {
      Cache.this.update(cached, network);
    }

    @Override public void trackConditionalCacheHit() {
      Cache.this.trackConditionalCacheHit();
    }

    @Override public void trackResponse(CacheStrategy cacheStrategy) {
      Cache.this.trackResponse(cacheStrategy);
    }
  };

  final DiskLruCache cache;
  public Cache(File directory, long maxSize) {
    this(directory, maxSize, FileSystem.SYSTEM);
  }

  Cache(File directory, long maxSize, FileSystem fileSystem) {
    this.cache = DiskLruCache.create(fileSystem, directory, VERSION, ENTRY_COUNT, maxSize);
  }
  ... ...
    
  @Nullable CacheRequest put(Response response) {
    String requestMethod = response.request().method();

    if (HttpMethod.invalidatesCache(response.request().method())) {
      try {
        remove(response.request());
      } catch (IOException ignored) {
        // The cache cannot be written.
      }
      return null;
    }
    if (!requestMethod.equals("GET")) {
      // Don't cache non-GET responses. We're technically allowed to cache
      // HEAD requests and some POST requests, but the complexity of doing
      // so is high and the benefit is low.
      return null;
    }

    if (HttpHeaders.hasVaryAll(response)) {
      return null;
    }

    Entry entry = new Entry(response);
    DiskLruCache.Editor editor = null;
    try {
      editor = cache.edit(key(response.request().url()));
      if (editor == null) {
        return null;
      }
      entry.writeTo(editor);
      return new CacheRequestImpl(editor);
    } catch (IOException e) {
      abortQuietly(editor);
      return null;
    }
  }
}

```

从他的构造方法就不难看出,`cache`内部使用`DiskLruCache`负责储存和读取文件,然后外部的`CacheInterceptor` 通过调用`internalCache`,来实现对缓存的存储和获取

`put`方法重点看一下

**Don't cache non-GET responses. We're technically allowed to cache   HEAD requests and some POST requests, but the complexity of doing   so is high and the benefit is low.**

不要缓存非GET响应。 从技术上讲，我们可以缓存HEAD请求和一些POST请求，但是这样做很复杂如此之高，而收益却很低。

**Okhttp Cache是不支持GET方法以外的缓存的**

### CacheControl

先看看`CacheControl`的字段都有哪些

```java
  private final boolean noCache;
  private final boolean noStore;
  private final int maxAgeSeconds;
  private final int sMaxAgeSeconds;
  private final boolean isPrivate;
  private final boolean isPublic;
  private final boolean mustRevalidate;
  private final int maxStaleSeconds;
  private final int minFreshSeconds;
  private final boolean onlyIfCached;
  private final boolean noTransform;
  private final boolean immutable;


```

我们实现缓存是通过 http header来实现的,手动写起来繁琐且容易出错,所以通过这个CacheControl来实现缓存相关的header解析和拼接.

- 拼接

  ```java
  private String headerValue() {
    StringBuilder result = new StringBuilder();
    if (noCache) result.append("no-cache, ");
    if (noStore) result.append("no-store, ");
    if (maxAgeSeconds != -1) result.append("max-age=").append(maxAgeSeconds).append(", ");
    if (sMaxAgeSeconds != -1) result.append("s-maxage=").append(sMaxAgeSeconds).append(", ");
    if (isPrivate) result.append("private, ");
    if (isPublic) result.append("public, ");
    if (mustRevalidate) result.append("must-revalidate, ");
    if (maxStaleSeconds != -1) result.append("max-stale=").append(maxStaleSeconds).append(", ");
    if (minFreshSeconds != -1) result.append("min-fresh=").append(minFreshSeconds).append(", ");
    if (onlyIfCached) result.append("only-if-cached, ");
    if (noTransform) result.append("no-transform, ");
    if (immutable) result.append("immutable, ");
    if (result.length() == 0) return "";
    result.delete(result.length() - 2, result.length());
    return result.toString();
  }
  ```

- 解析

  ```jav
  /**
     * Returns the cache directives of {@code headers}. This honors both Cache-Control and Pragma
     * headers if they are present.
     */
    public static CacheControl parse(Headers headers) {
      boolean noCache = false;
      boolean noStore = false;
      int maxAgeSeconds = -1;
      int sMaxAgeSeconds = -1;
      boolean isPrivate = false;
      boolean isPublic = false;
      boolean mustRevalidate = false;
      int maxStaleSeconds = -1;
      int minFreshSeconds = -1;
      boolean onlyIfCached = false;
      boolean noTransform = false;
      boolean immutable = false;
  
      boolean canUseHeaderValue = true;
      String headerValue = null;
  
      for (int i = 0, size = headers.size(); i < size; i++) {
        String name = headers.name(i);
        String value = headers.value(i);
  
        if (name.equalsIgnoreCase("Cache-Control")) {
          if (headerValue != null) {
            // Multiple cache-control headers means we can't use the raw value.
            canUseHeaderValue = false;
          } else {
            headerValue = value;
          }
        } else if (name.equalsIgnoreCase("Pragma")) {
          // Might specify additional cache-control params. We invalidate just in case.
          canUseHeaderValue = false;
        } else {
          continue;
        }
  
        int pos = 0;
        while (pos < value.length()) {
          int tokenStart = pos;
          pos = HttpHeaders.skipUntil(value, pos, "=,;");
          String directive = value.substring(tokenStart, pos).trim();
          String parameter;
  
          if (pos == value.length() || value.charAt(pos) == ',' || value.charAt(pos) == ';') {
            pos++; // consume ',' or ';' (if necessary)
            parameter = null;
          } else {
            pos++; // consume '='
            pos = HttpHeaders.skipWhitespace(value, pos);
  
            // quoted string
            if (pos < value.length() && value.charAt(pos) == '\"') {
              pos++; // consume '"' open quote
              int parameterStart = pos;
              pos = HttpHeaders.skipUntil(value, pos, "\"");
              parameter = value.substring(parameterStart, pos);
              pos++; // consume '"' close quote (if necessary)
  
              // unquoted string
            } else {
              int parameterStart = pos;
              pos = HttpHeaders.skipUntil(value, pos, ",;");
              parameter = value.substring(parameterStart, pos).trim();
            }
          }
  
          if ("no-cache".equalsIgnoreCase(directive)) {
            noCache = true;
          } else if ("no-store".equalsIgnoreCase(directive)) {
            noStore = true;
          } else if ("max-age".equalsIgnoreCase(directive)) {
            maxAgeSeconds = HttpHeaders.parseSeconds(parameter, -1);
          } else if ("s-maxage".equalsIgnoreCase(directive)) {
            sMaxAgeSeconds = HttpHeaders.parseSeconds(parameter, -1);
          } else if ("private".equalsIgnoreCase(directive)) {
            isPrivate = true;
          } else if ("public".equalsIgnoreCase(directive)) {
            isPublic = true;
          } else if ("must-revalidate".equalsIgnoreCase(directive)) {
            mustRevalidate = true;
          } else if ("max-stale".equalsIgnoreCase(directive)) {
            maxStaleSeconds = HttpHeaders.parseSeconds(parameter, Integer.MAX_VALUE);
          } else if ("min-fresh".equalsIgnoreCase(directive)) {
            minFreshSeconds = HttpHeaders.parseSeconds(parameter, -1);
          } else if ("only-if-cached".equalsIgnoreCase(directive)) {
            onlyIfCached = true;
          } else if ("no-transform".equalsIgnoreCase(directive)) {
            noTransform = true;
          } else if ("immutable".equalsIgnoreCase(directive)) {
            immutable = true;
          }
        }
      }
  
      if (!canUseHeaderValue) {
        headerValue = null;
      }
      return new CacheControl(noCache, noStore, maxAgeSeconds, sMaxAgeSeconds, isPrivate, isPublic,
          mustRevalidate, maxStaleSeconds, minFreshSeconds, onlyIfCached, noTransform, immutable,
          headerValue);
    }
   
  ```

### DiskLruCache

简单提一下`DiskLruCache`,应该是由Jake Wharton大神写的,后来也被Android官方认可,但是还没被放进Android API当中.官方地址[android.googlesource.com/platform/libcore/+/jb-mr2-release/luni/src/main/java/libcore/io/DiskLruCache.java](https://android.googlesource.com/platform/libcore/+/jb-mr2-release/luni/src/main/java/libcore/io/DiskLruCache.java).

但是Okhttp用的`DiskLruCache`是修改版的`DiskLruCache`,加入了okio的一些东西.具体有什么区别,感兴趣的可以看看源码.

DiskLruCache是用本地磁盘来做缓存,由于读写本地文件需要涉及IO流的操作，效率会稍低一些.它的缓存策略和`LruCache`一样,叫做**最近最少使用**的缓存策略.

### CacheInterceptor

Ohhttp是通过CacheInterceptor来实现缓存策略的,直接看源码吧

```java
@Override public Response intercept(Chain chain) throws IOException {
  	//获取request来获取本地缓存
    Response cacheCandidate = cache != null
        ? cache.get(chain.request())
        : null;

    long now = System.currentTimeMillis();
		//根据本地缓存和request来确定缓存策略
    CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();
    //如果networkRequest==null,说明可以用缓存
    Request networkRequest = strategy.networkRequest;
    //如果cacheResponse==null,说明缓存不可用
    Response cacheResponse = strategy.cacheResponse;

    if (cache != null) {
      cache.trackResponse(strategy);
    }

    if (cacheCandidate != null && cacheResponse == null) {
      closeQuietly(cacheCandidate.body()); // The cache candidate wasn't applicable. Close it.
    }
		
    //如果networkRequest==null,并且cacheResponse==null,那就是不能请求网络,页不能用缓存.
    //就会请求失败
    // If we're forbidden from using the network and the cache is insufficient, fail.
    if (networkRequest == null && cacheResponse == null) {
      return new Response.Builder()
          .request(chain.request())
          .protocol(Protocol.HTTP_1_1)
          .code(504)
          .message("Unsatisfiable Request (only-if-cached)")
          .body(Util.EMPTY_RESPONSE)
          .sentRequestAtMillis(-1L)
          .receivedResponseAtMillis(System.currentTimeMillis())
          .build();
    }
		
    //直接使用缓存,交给上级处理
    // If we don't need the network, we're done.
    if (networkRequest == null) {
      return cacheResponse.newBuilder()
          .cacheResponse(stripBody(cacheResponse))
          .build();
    }
		
   
    Response networkResponse = null;
    try {
      //缓存不可用,请求网络
      networkResponse = chain.proceed(networkRequest);
    } finally {
      // If we're crashing on I/O or otherwise, don't leak the cache body.
      if (networkResponse == null && cacheCandidate != null) {
        closeQuietly(cacheCandidate.body());
      }
    }
      // If we have a cache response too, then we're doing a conditional get.
  	//根据Response code,来判断使用缓存
    if (cacheResponse != null) {
      if (networkResponse.code() == HTTP_NOT_MODIFIED) {
        Response response = cacheResponse.newBuilder()
            .headers(combine(cacheResponse.headers(), networkResponse.headers()))
            .sentRequestAtMillis(networkResponse.sentRequestAtMillis())
            .receivedResponseAtMillis(networkResponse.receivedResponseAtMillis())
            .cacheResponse(stripBody(cacheResponse))
            .networkResponse(stripBody(networkResponse))
            .build();
        networkResponse.body().close();

        // Update the cache after combining headers but before stripping the
        // Content-Encoding header (as performed by initContentStream()).
        cache.trackConditionalCacheHit();
        cache.update(cacheResponse, response);
        return response;
      } else {
        closeQuietly(cacheResponse.body());
      }
    }

    Response response = networkResponse.newBuilder()
        .cacheResponse(stripBody(cacheResponse))
        .networkResponse(stripBody(networkResponse))
        .build();
		
  	//把获取到的缓存保存下来
    if (cache != null) {
      if (HttpHeaders.hasBody(response) && CacheStrategy.isCacheable(response, networkRequest)) {
        // Offer this request to the cache.
        CacheRequest cacheRequest = cache.put(response);
        return cacheWritingResponse(cacheRequest, response);
      }

      if (HttpMethod.invalidatesCache(networkRequest.method())) {
        try {
          cache.remove(networkRequest);
        } catch (IOException ignored) {
          // The cache cannot be written.
        }
      }
    }

    return response;
  }
```

这个缓存策略基本上和上面说的缓存基本类似,但是对header多了一项`immutable`判断,如果缓存中包含这个字段,也是直接使用缓存.

## OkHttp的连接池





























