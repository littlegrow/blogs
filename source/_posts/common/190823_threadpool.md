---
title: JAVA线程池
date: 2019-8-23 19:00:00
description: "JAVA线程池基础知识及常见问题总结"
categories:
- 线程池
---

### 线程池相关类
<br>

Eexecutor为灵活且强大的异步执行框架，其支持多种不同类型的任务执行策略，提供了一种标准的方法将任务的提交过程和执行过程解耦开发，基于生产者-消费者模式，其提交任务的线程相当于生产者，执行任务的线程相当于消费者，并用Runnable来表示任务，Executor的实现还提供了对生命周期的支持，以及统计信息收集，应用程序管理机制和性能监视等机制。

![](https://haitao.nos.netease.com/20190823162944_5eb2af66-188f-40e5-98fd-015c6a90e55b.jpg)

**Executor**：一个接口，其定义了一个接收Runnable对象的方法executor，其方法签名为executor(Runnable command),

**ExecutorService**：是一个比Executor使用更广泛的子类接口，其提供了生命周期管理的方法，以及可跟踪一个或多个异步任务执行状况返回Future的方法
 
**AbstractExecutorService**：ExecutorService执行方法的默认实现
 
**ScheduledExecutorService**：一个可定时调度任务的接口

**ThreadPoolExecutor**：线程池，可以通过调用Executors以下静态工厂方法来创建线程池并返回一个ExecutorService对象：

**ScheduledThreadPoolExecutor**：ScheduledExecutorService的实现，一个可定时调度任务的线程池

<br>

### ThreadPoolExecutor类

<br>

java.uitl.concurrent.ThreadPoolExecutor类是线程池中最核心的类，先看下这个类的构造函数：

```
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.acc = System.getSecurityManager() == null ?
                null :
                AccessController.getContext();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```

参数含义：

```
corePoolSize: 核心线程数，如果正在运行的线程小于这个值，则创建新线程来执行任务，即使线程池中有空闲的线程。
maximumPoolSize: 允许创建的最大线程数
keepAliveTime: 线程数多余corePoolSize时，多处线程的空闲等待最长时间
unit：空闲等待时间的单位
workQueue：用来存放待执行任务的队列
threadFactory：线程创建工厂，用于创建线程执行任务
handler：等待线程队列满时的拒绝策略
```

<br>

#### unit

<br>

    TimeUnit.DAYS;               //天
    TimeUnit.HOURS;             //小时
    TimeUnit.MINUTES;           //分钟
    TimeUnit.SECONDS;           //秒
    TimeUnit.MILLISECONDS;      //毫秒
    TimeUnit.MICROSECONDS;      //微妙
    TimeUnit.NANOSECONDS;       //纳秒

<br>

#### workQueue

<br>

    ArrayBlockingQueue：基于数组的先进先出队列，此队列创建时必须指定大小；
    LinkedBlockingQueue：基于链表的先进先出队列，如果创建时没有指定此队列大小，则默认为Integer.MAX_VALUE；
    synchronousQueue：这个队列比较特殊，它不会保存提交的任务，而是将直接新建一个线程来执行新来的任务。

<br>

#### handler

<br>

    ThreadPoolExecutor.AbortPolicy:丢弃任务并抛出RejectedExecutionException异常。
    ThreadPoolExecutor.DiscardPolicy：丢弃任务，但是不抛出异常。
    ThreadPoolExecutor.DiscardOldestPolicy：丢弃队列最前面的任务，然后重新尝试执行任务
    ThreadPoolExecutor.CallerRunsPolicy：由调用线程处理该任务

<br>

### 常用线程池

<br>

#### newFixedThreadPool

<br>

```
    public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }
```

<br>

#### newSingleThreadExecutor

<br>

```
    public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
```

<br>

#### newCachedThreadPool

<br>

```
    public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
```

<br>

#### newSingleThreadScheduledExecutor

<br>

```
    public static ScheduledExecutorService newSingleThreadScheduledExecutor(ThreadFactory threadFactory) {
        return new DelegatedScheduledExecutorService
            (new ScheduledThreadPoolExecutor(1, threadFactory));
    }
```

<br>

### 问题

#### execute() 和 submit() 区别

```
    /**
     * Executes the given command at some time in the future.  The command
     * may execute in a new thread, in a pooled thread, or in the calling
     * thread, at the discretion of the {@code Executor} implementation.
     *
     * @param command the runnable task
     * @throws RejectedExecutionException if this task cannot be
     * accepted for execution
     * @throws NullPointerException if command is null
     */
    void execute(Runnable command);

    /**
     * Submits a value-returning task for execution and returns a
     * Future representing the pending results of the task. The
     * Future's {@code get} method will return the task's result upon
     * successful completion.
     *
     * <p>
     * If you would like to immediately block waiting
     * for a task, you can use constructions of the form
     * {@code result = exec.submit(aCallable).get();}
     *
     * <p>Note: The {@link Executors} class includes a set of methods
     * that can convert some other common closure-like objects,
     * for example, {@link java.security.PrivilegedAction} to
     * {@link Callable} form so they can be submitted.
     *
     * @param task the task to submit
     * @param <T> the type of the task's result
     * @return a Future representing pending completion of the task
     * @throws RejectedExecutionException if the task cannot be
     *         scheduled for execution
     * @throws NullPointerException if the task is null
     */
    <T> Future<T> submit(Callable<T> task);

```


    接收的参数不一样
    submit()有返回值，而execute()没有;
    submit()可以进行Exception处理



<br>

#### 如何合理配置线程池的大小

<br>

一般需要根据任务的类型来配置线程池大小：
如果是CPU密集型任务，就需要尽量压榨CPU，参考值可以设为 NCPU+1
如果是IO密集型任务，参考值可以设置为2*NCPU
这只是一个参考值，具体的设置还需要根据实际情况进行调整，比如可以先将线程池大小设置为参考值，再观察任务运行情况和系统负载、资源利用率来进行适当调整。

<br>

#### 线程池里线程如何复用

<br>

Thread.start()只能调用一次，一旦这个调用结束，则该线程就到了stop状态，不能再次调用start，要达到复用的目的，必须从Runnable接口的run()方法上入手，它本质上是个无限循环，跑的过程中不断检查我们是否有新加入的子Runnable对象。

首先看下执行线程任务 `runWoker()`，截取代码片段：

```
final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            while (task != null || (task = getTask()) != null) {
                w.lock();

                ......

        } finally {
            processWorkerExit(w, completedAbruptly);
        }
    }
```

重点关注一下 `getTask()` 方法：

```
private Runnable getTask() {
        
        ......

            try {
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }

        ......

    }
```

从等待队列里去取待执行的任务，只要能取到任务，线程就不会退出。


<br><br><br><br>
