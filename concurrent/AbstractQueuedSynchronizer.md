AQS(AbstractQueuedSynchronizer)是 java.util.concurrent的基础。J.U.C中宣传的封装良好的同步工具类Semaphore、CountDownLatch、ReentrantLock、ReentrantReadWriteLock、FutureTask等虽然各自都有不同特征，但是简单看一下源码，每个类内部都包含一个如下的内部类定义

``` 
abstract static class Sync extends AbstractQueuedSynchronizer
```

http://www.cnblogs.com/leesf456/p/5350186.html
http://brokendreams.iteye.com/blog/2250372
http://blog.csdn.net/yuenkin/article/details/50867530
http://ifeve.com/introduce-abstractqueuedsynchronizer/

Lock、ReentrantLock和AbstractQueuedSynchronizer的源码要点分析整理
Condition及其队列操作是基于AQS实现的
http://www.molotang.com/articles/480.html

AbstractQueuedSynchronizer在工具类Semaphore、CountDownLatch、ReentrantLock中的应用和CyclicBarrier
http://www.molotang.com/articles/487.html