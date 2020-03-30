# OkHttp Part2  配置中心-OkHttpClient

OkHttpClient就是OkHttp的配置中,它的每个成员变量就对应一个配置.我们分别来看一下都有哪些成员变量,它们的作用分别是什么

```java
/**管理请求的调度器,内部通过ExecutorService和maxRequest一起来实现请求策略,后面会专门写文章介绍*/
final Dispatcher dispatcher;
/**代理 比如国内访问不到google,可以先代理到可以访问google的服务器,VPN*/
final @Nullable Proxy proxy;
/**客户端所支持的协议的版本 例如HTTP1.O HTTP1.1 HTTP2.O*/
final List<Protocol> protocols;
/**客户端接受的Https加密套件是什么,TSL版本*/
final List<ConnectionSpec> connectionSpecs;
/**开发者自定义的interceptor*/
final List<Interceptor> interceptors;
/**开发者自定义的interceptor,处理请求和返回的位置和interceptors不一样.*/
final List<Interceptor> networkInterceptors;
/**请求的监听器,主要监听请求的数量,大小和持续时间*/
final EventListener.Factory eventListenerFactory;
/** */
final ProxySelector proxySelector;
/**可以用于添加和存储Cookie*/
final CookieJar cookieJar;
/**内部通过internalCache来实现缓存,这个缓存是符合Http缓存策略的缓存,不是类似无网条件下的那种缓存*/
final @Nullable Cache cache;
/**真正实现缓存的类*/
final @Nullable InternalCache internalCache;
/**通过地址和端口号创建Socket,通过Socket对服务器进行listen，accept，connect，read和write等等。*/
final SocketFactory socketFactory;
/**SSL socket*/>
final SSLSocketFactory sslSocketFactory;
/**将服务器返回的原始证书信息整理,第一个证书是访问服务器的证书,最后的证书是受信任的CA证书
 * 以省略与TLS握手无关的意外证书，并提取受信任的CA证书,以使证书更容易使用
 */
final CertificateChainCleaner certificateChainCleaner;
/**验证返回证书的host是否是要访问的host*/
final HostnameVerifier hostnameVerifier;
/** 
 * 用于三方自签名证书,因为是自签名证书,所以本地根证书对证书验证肯定会不过.所以要自己验证.
 * 这种验证方式好处是很简单,拿到证书公钥就可以进行验证了.也不用把证书额外放到客户端中.
 * 使用方法,CertificatePinner类注释上有介绍,就不多提了
 */
final CertificatePinner certificatePinner;
/**
 * Authorization登录授权相关.
 * 没有登录返回的code是401,RetryAndFollowUpInterceptor会调用authenticator.authenticate()
 * 如果返回的是407,RetryAndFollowUpInterceptor会调用proxyAuthenticator.authenticate().
 */
final Authenticator proxyAuthenticator;
final Authenticator authenticator;
/** 连接池,后面有专门写文章介绍*/
final ConnectionPool connectionPool;
/**DNS解析,通过域名拿到IP列表*/
final Dns dns;
/**
 * HTTP CODE 300-303 重定向
 * followSslRedirects 当访问的是HTTPs,让你跳HTTP的时候,是否跳转.或者HTTP->HTTPS
 * followRedirects,是否需要跳转
 * 默认值都是true
 */
final boolean followSslRedirects;
final boolean followRedirects;
//连接失败是否重新建立连接
final boolean retryOnConnectionFailure;
//请求之前存活时间.0表示一直存活
final int callTimeout;
// TCP连接时间
final int connectTimeout;
// 读取Response的时候
final int readTimeout;
// 写入Request的时间
final int writeTimeout;
// 发送长连接心跳包的时间间隔
final int pingInterval;
```

