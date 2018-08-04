---
title: "线程池的使用总结"
date: 2016-03-07 20:08:50 +0800
tags:
 - 线程
---

本博文是对Java线程池使用的一篇总结，系统记录下线程池的用法：

## 为什么使用线程池，线程池的好处是什么？

1. 相比于每次都创建新的Thread，通过重用线程池中的线程，减少了创建线程和销毁线程带来的性能开销。

2. 对线程进行管理控制，控制线程并发数量、定时执行、间隔执行等。

<!-- more -->

## Java线程池模型的UML类图

![](/images/articles/threadpool.png)

其中 **ThreadPoolExecutor** 是线程池的真正实现，从其构造方法中可以看出其创建的细节和需要配置的参数

``` java

public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {...}

```

下面对每个参数进行解释：

- **corePoolSize**: 线程池的核心线程数，默认情况下创建的核心线程不会被销毁，除非设置ThreadPoolExecutor#allowCoreThreadTimeOut为true,则根据keepAliveTime指定的超时时长进行销毁。

- **maximumPoolSize**: 线程池容纳的最大线程数（*除了核心线程，剩下的为非核心线程*），线程池中运行的线程不能超过此数量。

- **keepAliveTime**: 非核心线程闲置时的超时时长，超过此时间，非核心线程会被销毁，如果设置ThreadPoolExecutor#allowCoreThreadTimeOut为true，这核心线程也享有此特性。

- **unit**: 超时时长的时间单位。

- **workQueue**: 缓存的任务队列，当添加的任务数量大于corePoolSize时，任务将会被添加到此队列中进行缓存。

- **threadFactory**:线程池用来创建线程的工厂类，只有一个**newThread(Runnable r)** 的接口方法。

- **RejectedExecutionHandler**: 拒绝策略。当线程池中的线程达到最大线程数并且缓存任务队列已满，会调用此对象来进行处理。

RejectedExecutionHandler是一个接口，

```java
public interface RejectedExecutionHandler {
    void rejectedExecution(Runnable r, ThreadPoolExecutor executor);
}
```

有四个实现类，对应4种处理策略，从源码中可以看出他们的作用：

**AbortPolicy** 丢弃当前添加的任务并且抛出RejectedExecutionException异常

**DiscardPolicy** 丢弃当前添加的任务，但是不抛出异常

**DiscardOldestPolicy** 丢弃最老的任务，也就是任务队列队首的任务，然后再尝试执行当前添加的任务

**CallerRunsPolicy** 由调用线程处理该任务

## 线程池的执行策略：

1. 如果线程池中的线程数量<核心线程数量，直接创建一个核心线程执行该任务。

2. 如果线程池中的线程数量>=核心线程数量，则将任务插入到缓存任务队列中，如上 **workQueue** 参数指定的队列中。

3. 如果缓存任务队列已满不能继续添加任务，并且线程池中的线程数量<最大线程数，则创建一个非核心线程来执行该任务。

4. 如果缓存任务队列和线程池均已满，则调用 **RejectedExecutionHandler#rejectedExecution** 进行处理。

**另外：**

- 如果线程池中的线程执行任务完毕从而闲置，则从缓存任务队列头取出一个任务，继续进行执行。（**优点一，重用已经创建的线程**）

- 如果线程池中线程闲置，并且缓存队列中已经没有任务，则线程池会根据超时规则进行线程的销毁。一般情况下，核心线程不进行销毁，非核心线程根据设置的超时间长进行销毁。

- 处理任务的优先级： 核心线程池 > 缓存任务队列 > 非核心线程池


## 常用的4种线程池模型：

**Executors** 提供了几个静态工厂方法来创建几种不同特性的线程池，本质上就是通过配置ThreadPoolExecutor构造方法的参数组合，实现具有不同特性的ThreadPool。

#### 1、FixedThreadPool - 固定线程数量的线程池
```java

public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}

```

从ThreadPoolExecutor的构造方法参数中可以看出，FixedThreadPool 中只有核心线程。当核心线程用完后，新添加的任务进入任务队列**new LinkedBlockingQueue<Runnable>()** 中：

```java
public LinkedBlockingQueue() {
    this(Integer.MAX_VALUE);
}
```

默认的构造方法创建了一个无限大的队列，所以我们可以添加任意数目的任务，但是最大并发数只是*nThreads*。

前面说过，默认情况下，核心线程是不会销毁的，即使它们执行完任务变为空闲状态，这样带来的好处是可以快速响应之后添加的新任务（免去了创建线程的性能开销）。

但是可能带来一些意想不到的问题，比如**内存泄露**。之前在进行Andorid项目的性能优化时就遇到过线程池带来的内存泄露，正是由于线程池中的核心线程持有了外部的对象（View、Activity等）,并且自身不会主动销毁，导致Activity等持有的其他大对象迟迟不能被GC回收，内存泄露。

当然，有解决方案：

**ThreadPoolExecutor#shutdown()** 关闭线程池。

 调用此方法后，ExecutorService不会立即关闭，但是不再接受新的任务，直到所有线程都执行完毕才会关闭，所有在shutdown执行前提交的任务都会得到执行。

**ThreadPoolExecutor#shutdownNow()** 立即关闭线程池。

调用此方法，会尝试关闭所有正在执行的任务（但是不能做任何的保证，它们可能都停止，也可能都完成），跳过正在等待的任务。


最后，使用弱引用的方式持有外部对象也是一个保险的做法。


#### 2、CachedThreadPool - 线程数量不定的线程池

```java
public static ExecutorService newCachedThreadPool() {
   return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                 60L, TimeUnit.SECONDS,
                                 new SynchronousQueue<Runnable>());
}
```

同样，从构造方法参数中可以看出，CachedThreadPool没有核心线程池，最大线程数量为无限大，也就是可以并发执行任意数目的线程（超时销毁时长为60s)。

其中 **new SynchronousQueue<Runnable>()** 是一个特殊的任务队列，可以理解为一个管道，只能通过任务，不能存储任何任务，这就导致所有插入的任务都会立即得到执行。

从此线程池的特性看来，这类线程池比较适合执行大量的耗时较少的任务。一方面，线程的并发数量是无限的。另一方面，60s的超时时长保证闲置的线程销毁，最终整个线程池不包含任何线程，不占用系统宝贵资源。

#### 3、SingleThreadExecutor - 只有一个核心线程的线程池

```java
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
```
跟FixedThreadPool类似，只是**nThreads**固定为1，算是它的一个特例。确保了所有任务都在同一个线程按顺序同步执行，这里不再多述。

#### 4、ScheduledThreadPool - 进行定时以及周期性任务执行的线程池

```java
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
    return new ScheduledThreadPoolExecutor(corePoolSize);
}
```
```java
public ScheduledThreadPoolExecutor(int corePoolSize) {
    super(corePoolSize, Integer.MAX_VALUE,
          DEFAULT_KEEPALIVE_MILLIS, MILLISECONDS,
          new DelayedWorkQueue());
}
```
从UML类图中，可以看出此类线程池的特殊之处-实现了**ScheduledExecutorService**接口，此接口中主要定义如下几个接口方法：

```java
/**
 * 延时delay后执行command
 */
public ScheduledFuture<?> schedule(Runnable command, long delay, TimeUnit unit);

public <V> ScheduledFuture<V> schedule(Callable<V> callable, long delay, TimeUnit unit);

/**
 * 初始延时initialDelay后，每隔period执行一次command
 */
public ScheduledFuture<?> scheduleAtFixedRate(Runnable command, long initialDelay, long period, TimeUnit unit);

public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command, long initialDelay,long delay,TimeUnit unit);
```

使用此类线程池可以定时执行任务和固定周期重复执行任务。
