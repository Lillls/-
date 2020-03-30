# OkHttp Part3  请求策略和线程模型

之前说过:Dispatcher是管理请求的调度器,内部通过ExecutorService和maxRequest一起来实现请求策略,这篇文章详细介绍一下OkHttp的请求策略和线程模型

## 线程模型

