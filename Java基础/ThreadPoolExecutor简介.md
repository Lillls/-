# 线程池ThreadPoolExecutor简介

## 线程池的实现方式

java中实现线程池的类主要是`ThreadPoolExecutor`这个类

主要的构造方法有

![image-20191119172740407](http://xiaoyu-ipic.oss-cn-beijing.aliyuncs.com/blog/2019-11-19-092740.png)

来看一下最全的构造方法

```java
 public ThreadPoolExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit, BlockingQueue<Runnable> workQueue, ThreadFactory threadFactory, RejectedExecutionHandler handler)
```

来介绍一下它的参数

1. **`int corePoolSize`**

   核心线程数,默认情况下核心线程会一直存活,即使处于闲置状态.但是如果将`allowCoreThreadTimeOut`设置为true的话,那么闲置的核心线程也会和非核心线程一样有超时策略.这个超时值由`keepAliveTime`指定.

2. **`int maximumPoolSize`**

   最大线程数

3. **`long keepAliveTime`**

   非核心线程数的存活时间.如果非核心线程处于闲置状态,超过这个时长就会被回收.

4. **`TimeUnit unit`**

   存活时间的时间单位

5. **`BlockingQueue<Runnable> workQueue`**

   任务队列,当线程池创建的线程数量大于 `corePoolSize` 后，新来的任务将会加入到堵塞队列（workQueue）中,等待有空闲线程来执行.常用的有:	

   1. LinkedBlockingQueue  默认长度是 `Integer.MAX_VALUE`也可以通过构造方法设置一个长度.
   2. SynchronousQueue     以一个很特殊的队列. 它不会保存任务，每一个新增任务的线程必须等待另一个线程取出任务,也可以把它看成容量为0的队列.简单来说就是有了任务之后直接提交给thread运行.

6. **`ThreadFactory threadFactory`**

   `ThreadFactory`是一个接口,用于创建新的线程

   ```java
   public interface ThreadFactory {
       Thread newThread(Runnable r);
   }
   ```

7. **`RejectedExecutionHandler handler`**

   `RejectedExecutionHandler`也是一个接口.当线程池已满或者线程队列已满时,会调用rejectedExecution实行拒绝策略

   ```java
   public interface RejectedExecutionHandler {
       void rejectedExecution(Runnable r, ThreadPoolExecutor executor);
   }
   ```

   有为四种策略

   1. AbortPolicy                  直接抛出 RejectedExecutionException 异常。
   2. CallerRunsPolicy          会在运行线程池的线程中处理被拒绝的任务。
   3. DiscardOldestPolicy    放弃等待队列中最旧的未处理任务，然后将被拒绝的任务添加到等待队列中。
   4. DiscardPolicy                线程池将丢弃被拒绝的任务。

   默认是AbortPolicy,直接抛出 RejectedExecutionException 异常。

## 线程池的execute规则

先来看看源码

```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    /*
     * Proceed in 3 steps:
     *
     * 1. If fewer than corePoolSize threads are running, try to
     * start a new thread with the given command as its first
     * task.  The call to addWorker atomically checks runState and
     * workerCount, and so prevents false alarms that would add
     * threads when it shouldn't, by returning false.
     *
     * 2. If a task can be successfully queued, then we still need
     * to double-check whether we should have added a thread
     * (because existing ones died since last checking) or that
     * the pool shut down since entry into this method. So we
     * recheck state and if necessary roll back the enqueuing if
     * stopped, or start a new thread if there are none.
     *
     * 3. If we cannot queue task, then we try to add a new
     * thread.  If it fails, we know we are shut down or saturated
     * and so reject the task.
     */
    int c = ctl.get();
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        if (! isRunning(recheck) && remove(command))
            reject(command);
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    else if (!addWorker(command, false))
        reject(command);
}
```

1. 当线程数**小于`corePoolSize`**时,会直接启动一个线程来执行任务.这个线程也叫核心线程.
2. 当线程数**大于等于`corePoolSize`**时,会将任务插入到任务队列中.
3. 当任务队列为**SynchronousQueue**,因为他的容量为(其实是不保存任务),会直接启动一个非核心线程来执行.
4. 或者当任务队列已满时,也会直接启动一个非核心线程来执行
5. 如果线程数**大于maximumPoolSize**,就会执行拒绝策略.调用`handler.rejectedExecution`

## 怎么使用线程池

先说说禁止怎么使用吧

根据***阿里巴巴Android开发手册***

- 【强制】新建线程时，**必须通过线程池提供(AsyncTask 或者 ThreadPoolExecutor** 或者其他形式自定义的线程池)，不允许在应用中自行显式创建线程。 

  说明:

  使用线程池的好处是减少在创建和销毁线程上所花的时间以及系统资源的开销，解决资源不足的问题。如果不使用线程池，有可能造成系统创建大量同类线程而导致消耗完内存或者“过度切换”的问题。另外创建匿名线程不便于后续的资源使用分析， 对性能分析等会造成困扰。 

- 【强制】线程池不允许使用 Executors 去创建，而是通过 ThreadPoolExecutor 的方 式，这样的处理方式让写的同学更加明确线程池的运行规则，规避资源耗尽的风险。 

  说明:

  Executors 返回的线程池对象的弊端如下: 

  1. FixedThreadPool 和 SingleThreadPool : 允 许 的 请 求 队 列 长 度 为 Integer.MAX_VALUE，可能会堆积大量的请求，从而导致 OOM; 
  2. CachedThreadPool 和 ScheduledThreadPool : 允 许 的 创 建 线 程 数 量 为 Integer.MAX_VALUE，可能会创建大量的线程，从而导致 OOM。 

正例:

```java
int NUMBER_OF_CORES = Runtime.getRuntime().availableProcessors();
int KEEP_ALIVE_TIME = 1;
TimeUnit KEEP_ALIVE_TIME_UNIT = TimeUnit.SECONDS;
BlockingQueue<Runnable> taskQueue = new LinkedBlockingQueue<Runnable>(); 
ExecutorService executorService = new ThreadPoolExecutor(NUMBER_OF_CORES, NUMBER_OF_CORES*2, KEEP_ALIVE_TIME, KEEP_ALIVE_TIME_UNIT,taskQueue, new BackgroundThreadFactory(), new DefaultRejectedExecutionHandler());
```

我觉得LinkedBlockingQueue应该设置一个长度比较好.

可以模仿AsyncTask中对线程池的使用.

```java
int CPU_COUNT = Runtime.getRuntime().availableProcessors();
int CORE_POOL_SIZE = Math.max(2, Math.min(CPU_COUNT - 1, 4));
int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;
int KEEP_ALIVE_SECONDS = 30;

final ThreadFactory threadFactory = new ThreadFactory() {
  private final AtomicInteger mCount = new AtomicInteger(1);

  public Thread newThread(Runnable r) {
    return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());
  }
};

final BlockingQueue<Runnable> poolWorkQueue =
  new LinkedBlockingQueue<>(128);

ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
  CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE_SECONDS, TimeUnit.SECONDS,
  poolWorkQueue, threadFactory);
```

## CAS

CAS ABA的问题.

因为CAS需要在操作值的时候检查下值有没有发生变化，如果没有发生变化则更新，但是如果一个值原来是A，变成了B，又变成了A，那么使用CAS进行检查时会发现它的值没有发生变化，但是实际上却变化了。ABA问题的解决思路就是使用版本号。在变量前面追加上版本号，每次变量更新的时候把版本号加一，那么A－B－A 就会变成1A-2B－3A。

从Java1.5开始JDK的atomic包里提供了一个类AtomicStampedReference来解决ABA问题。这个类的compareAndSet方法作用是首先检查当前引用是否等于预期引用，并且当前标志是否等于预期标志，如果全部相等，则以原子方式将该引用和该标志的值设置为给定的更新值。

线程1准备用CAS将变量的值由A替换为B，在此之前，线程2将变量的值由A替换为C，又由C替换为A，然后线程1执行CAS时发现变量的值仍然为A，所以CAS成功。但实际上这时的现场已经和最初不同了，尽管CAS成功，但可能存在潜藏的问题.

