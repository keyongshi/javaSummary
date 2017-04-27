## Java中的线程池

我们的工作中需要处理很多任务，大多数情况下使用线程池，可以提高程序的性能，合理使用线程池可以带来3个好处

- 降低资源消耗。
- 提高响应速度。
- 提高线程的可管理性。



而说起java中的多线程，不得不提的就是ThreadPoolExecutor，也就是最常说的线程池，其实现了Executor接口
```
public interface Executor {

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
}
```
Executor基于生产者-消费者模式，提交任务的线程相当于生产者，处理任务的线程相当于消费者。
### 构造函数
首先来看下其构造函数
```java
    /**
     * Creates a new {@code ThreadPoolExecutor} with the given initial
     * parameters.
     *
     * @param corePoolSize the number of threads to keep in the pool, even
     *        if they are idle, unless {@code allowCoreThreadTimeOut} is set
     * @param maximumPoolSize the maximum number of threads to allow in the
     *        pool
     * @param keepAliveTime when the number of threads is greater than
     *        the core, this is the maximum time that excess idle threads
     *        will wait for new tasks before terminating.
     * @param unit the time unit for the {@code keepAliveTime} argument
     * @param workQueue the queue to use for holding tasks before they are
     *        executed.  This queue will hold only the {@code Runnable}
     *        tasks submitted by the {@code execute} method.
     * @param threadFactory the factory to use when the executor
     *        creates a new thread
     * @param handler the handler to use when execution is blocked
     *        because the thread bounds and queue capacities are reached
     * @throws IllegalArgumentException if one of the following holds:<br>
     *         {@code corePoolSize < 0}<br>
     *         {@code keepAliveTime < 0}<br>
     *         {@code maximumPoolSize <= 0}<br>
     *         {@code maximumPoolSize < corePoolSize}
     * @throws NullPointerException if {@code workQueue}
     *         or {@code threadFactory} or {@code handler} is null
     */
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
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```

### 参数解读
从构造函数的参数中，我们看到了以下参数，再结合注释，我们能得到大量信息
- corePoolSize：核心线程数，当提交一个任务到线程池时，如果线程池中的线程数量小于corePoolSize，即使其他空闲的基本线程能够执行新任务也会创建线程，线程池会创建一个线程来执行任务。
- maximumPoolSize：最大线程数，如果队列满了，并且线程数小于最大线程数，则线程池会再创建新的线程执行任务。
- keepAliveTime：线程存活时间，即超过了核心线程数之后，线程被销毁之前的空闲时间
- TimeUnit：线程存活时间单位
- BlockingQueue：任务执行前的存储队列，该队列为阻塞队列，可以选择ArrayBlockingQueue, LinkedBlockingQueue, SynchronusQueue, PriorityBlockingQueue，在后续的介绍中会讲到这几种队列。
- ThreadFactory：创建线程的工厂
- RejectedExecutionHandler：当任务数超过了存储队列的容量时的拒绝策略

### 任务提交时，ThreadPoolExecutor的变化
我们需要重点了解提交任务到ThreadPoolExecutor之后corePoolSize，maximumPoolSize，BlockingQueue和RejectedExecutionHandler是如何工作的

当一个新任务被提交给ThreadPoolExecutor的时候，处理流程大概是这样的:

- 如果当前线程池中线程的数目低于corePoolSize，则创建新线程执行提交的任务，而无需检查当前是否有空闲线程
- 如果提交任务时线程池中线程数已达到corePoolSize，则将提交的任务加入等待执行队列
- 如果提交任务时等待执行的任务队列是有限队列，而且已满，则在线程池中开辟新线程执行此任务
- 如果线程池中线程数目已达到maximumPoolSize，则提交的任务交由RejectedExecutionHandler处理

默认情况下，ThreadPoolExecutor的线程数是根据需求来延迟初始化的，即有新任务加进来的时候才会挨个创建线程。

当线程池中的线程个数超出了corePoolSize，那么空闲的线程一旦超出keepAliveTime的时间就会被终止，以节省系统资源。

ThreadPoolExecutor这么设计，是为了在执行execute()方式时，尽可能地避免获取全局锁，而大多数情况下，是将任务放入BlockingQueue中，利用BlockingQueue的特性来避免获取全局锁。

### 拒绝策略
RejectedExecutionHandler有以下4种策略选择，默认的策略是AbortPolicy
- AbortPolicy：抛出异常。抛出一个java.util.concurrent.RejectedExecutionException异常。
- DiscardPolicy：直接丢弃。将此次提交的任务丢弃，不做处理。使用该策略可能会出现任务悄无声息地扔掉了。
- DiscardOldestPolicy：丢弃最老的任务。该策略和DiscardPolicy类似，但丢弃的是队列中最老的任务。
- CallerRunsPolicy：调用者执行。任务交给提交任务的线程去执行。该策略既不会丢失任务，也不会抛出异常，而是让caller线程执行任务，一石二鸟，既执行了任务，还推迟了任务入队的速度。

### 线程工厂

可以通过线程工厂给每个创建的线程设置更具有意义的名字，例如使用开源框架Guava的ThreadFactoryBuilder可以给线程设置更有意义的名字。

### 给线程池提交任务

给线程池提交任务有两种方式，分别是execute()和submit()。

execute()方法用于提交不需要返回值的任务。

submit()方法用于提交需要返回值的任务。线程池会返回一个future类型的对象，可以通过future.get()获取返回值。

### 关闭线程池

通过调用shutdown来关闭线程池，通过遍历线程池中的工作线程，然后逐个调用线程的interrupt方法来中断线程，调用该方法后，isShutdown方法会返回true。

### 包装使用
Executors中一些工厂方法，可以方便地创建不同特性的线程池，也都是依赖ThreadPoolExecutor来实现的
- Executors.newCachedThreadPool() 线程数量无穷大的线程池，线程会自动重用。
  - 适用于执行很多的短期异步任务，适合负载比较轻的服务器。
  - corePoolSize设置为0，maximumPoolSize设置为Integer.MAX_VALUE，意味着核心线程数0，最大线程数是无穷。
  - keepAliveTime设置为60L，意味着线程空闲60s即被终止。
  - 使用没有容量的SynchronousQueue作为队列，即提交任务时，创建新新线程来处理任务
- Executors.newFixedThreadPool(int) 固定线程数的线程池，队列无穷大。
  - 适用于需要限制当前线程数量的应用场景，以控制资源，适合负载比较重的服务器。
  - 使用了无界的LinkedBlockingQueue作为存储队列。
- Executors.newSingleThreadExecutor() 单线程执行器，队列无穷大。
  - 适用于需要保证顺序地执行各个任务，并且在任意时间点，不会有多个线程的应用场景。
  - 使用了无界的LinkedBlockingQueue作为存储队列。
- Executors.newScheduledThreadPool(int)，依赖ScheduledThreadPoolExecutor，可以在给定的延迟后运行命令，或者定期执行命令。适用于需要多个后台线程执行周期任务，同时为了满足资源管理的需求而需要限制后台线程数量的应用场景。

```java
    /**
     * Creates a thread pool that reuses a fixed number of threads
     * operating off a shared unbounded queue.  At any point, at most
     * {@code nThreads} threads will be active processing tasks.
     * If additional tasks are submitted when all threads are active,
     * they will wait in the queue until a thread is available.
     * If any thread terminates due to a failure during execution
     * prior to shutdown, a new one will take its place if needed to
     * execute subsequent tasks.  The threads in the pool will exist
     * until it is explicitly {@link ExecutorService#shutdown shutdown}.
     *
     * @param nThreads the number of threads in the pool
     * @return the newly created thread pool
     * @throws IllegalArgumentException if {@code nThreads <= 0}
     */
    public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }
    /**
     * Creates an Executor that uses a single worker thread operating
     * off an unbounded queue. (Note however that if this single
     * thread terminates due to a failure during execution prior to
     * shutdown, a new one will take its place if needed to execute
     * subsequent tasks.)  Tasks are guaranteed to execute
     * sequentially, and no more than one task will be active at any
     * given time. Unlike the otherwise equivalent
     * {@code newFixedThreadPool(1)} the returned executor is
     * guaranteed not to be reconfigurable to use additional threads.
     *
     * @return the newly created single-threaded Executor
     */
    public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
    /**
     * Creates a thread pool that creates new threads as needed, but
     * will reuse previously constructed threads when they are
     * available.  These pools will typically improve the performance
     * of programs that execute many short-lived asynchronous tasks.
     * Calls to {@code execute} will reuse previously constructed
     * threads if available. If no existing thread is available, a new
     * thread will be created and added to the pool. Threads that have
     * not been used for sixty seconds are terminated and removed from
     * the cache. Thus, a pool that remains idle for long enough will
     * not consume any resources. Note that pools with similar
     * properties but different details (for example, timeout parameters)
     * may be created using {@link ThreadPoolExecutor} constructors.
     *
     * @return the newly created thread pool
     */
    public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
    /**
     * Creates a thread pool that can schedule commands to run after a
     * given delay, or to execute periodically.
     * @param corePoolSize the number of threads to keep in the pool,
     * even if they are idle
     * @return a newly created scheduled thread pool
     * @throws IllegalArgumentException if {@code corePoolSize < 0}
     */
    public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
        return new ScheduledThreadPoolExecutor(corePoolSize);
    }
```

任务处理相关源码分析
http://www.molotang.com/articles/522.html

生命周期相关源码分析
http://www.molotang.com/articles/526.html



### 设置线程池的大小

设置过大，大量的线程将在相对很少的CPU和内存上发生激烈竞争，不仅导致更高的内存使用量，并且任务处理速度也不理想。

设置过小，导致cpu空闲，降低了吞吐率

如果有N个处理器，线程池设置的数量

- 计算密集型任务，线程池大小设置为N+1
- IO密集型任务，线程池大小设置为2N

### 线程池的监控

可以通过以下属性监控线程池的使用情况

- taskCount：线程池需要执行的任务数量，包含未执行的和已执行的。
- completedTaskCount：线程池在运行过程中已完成的任务数量，小于等于taskCount。
- largestPoolSize：线程池曾经创建的最大线程数量，通过这个数据，可以知道线程池曾经是否满过。
- getPoolSize：线程池的线程数量。
- getActiveCount：获取活动的线程数。

