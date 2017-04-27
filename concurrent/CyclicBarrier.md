## CyclicBarrier的定义
## CyclicBarrier与CountDownLatch的区别
CyclicBarrier相对于CountDownLatch可以重用
## CyclicBarrier的应用场景
## CyclicBarrier的原理
CyclicBarrier 主要用 ReeantrantLock 与 Condition 来控制线程资源的获取。
构造函数时传入parties，表明参与到这次 barrier 的参与者个数，然后调用await方法时按照以下步骤工作

1. 一共有parties个线程要求它们都到达 barrier.await() 才能继续向下执行
2. 前几个线程调用 barrier.await() 时其实时在内部统一调用 Reeantrant.lock()获取 lock, 然后再调用 Condition.await() 将lock释放, 等待唤醒
3. 第parties个线程到达 barrier.await() 处, 先调用 Reeantrant.lock() 然后发现自己是最后一个线程, 则直接调用 Condition.signalAll() 唤醒其他线程, 最后自己释放 lock
4. 其他几个线程被 signal 了都争相获取 lock, 最后又释放
## CyclicBarrier源码解读
### 构造函数
```java
/**
 * 指定 barrierCommand 的构造 CyclicBarrier
 */
public CyclicBarrier(int parties, Runnable barrierCommand) {
    if(parties <= 0) throw new IllegalArgumentException();
    this.parties = parties;
    this.count = parties;
    this.barrierCommand = barrierCommand;
}

/**
 * 构造 CyclicBarrier
 */
public CyclicBarrier(int parties){
    this(parties, null);
}
```
### 主要属性
```java
private static class Generation{
    boolean broken = false;
}

/** The lock for guarding barrier entry */
/** 全局的重入 lock */
private final ReentrantLock lock = new ReentrantLock();
/** Condition to wait on until tripped */
/** 控制线程等待  */
private final Condition trip = lock.newCondition();
/** The number of parties */
/** 参与到这次 barrier 的参与者个数 */
private final int parties;
/** The command to run when tripped */
/** 到达 barrier 时执行的command */
private final Runnable barrierCommand;
/** The current generation */
/** 初始化 generation */
private Generation generation = new Generation();
```
### 生成 generation 方法
这是在 一个 barrier 完成后, 重新初始化值
```java
/**
 * Updates state on barrier trip and wakes up everyone.
 * Called only while holding lock.
 */
/** 生成下一个 generation */
private void nextGeneration(){
    // signal completion of last generation
    // 唤醒所有等待的线程来获取 AQS 的state的值
    trip.signalAll();
    // set up next generation
    // 重新赋值计算器
    count = parties;
    // 重新初始化 generation
    generation = new Generation();
}
```
### breakBarrier 方法
breakBarrier 主要用于等待的线程当被中断, 或等待超时执行
```java
/**
 * Sets current barrier generation as broken and wakes up everyone
 * Called only while holding lock
 */
/** 当某个线程被中断 / 等待超时 则将 broken = true, 并且唤醒所有等待中的线程 */
private void breakBarrier(){
    generation.broken = true;
    count = parties;
    trip.signalAll();
}
```
### 最重要的方法await方法
```java
/**
 * 进行等待所有线程到达 barrier
 * 除非: 其中一个线程被 inetrrupt
 */
public int await() throws InterruptedException, BrokenBarrierException{
    try{
        return dowait(false, 0L);
    }catch (TimeoutException toe){
        throw new Error(toe); // cannot happen
    }
}

/**
 * 进行等待所有线程到达 barrier
 * 除非: 等待超时
 */
public int await(long timeout, TimeUnit unit) throws Exception{
    return dowait(true, unit.toNanos(timeout));
}

/**
 * Main barrier code, covering the various policies
 */
/**
 * CyclicBarrier 的核心方法, 主要是所有线程都获取一个 ReeantrantLock 来控制
 */
private int dowait(boolean timed, long nanos)throws InterruptedException, BrokenBarrierException, TimeoutException{
    final ReentrantLock lock = this.lock;
    lock.lock();                            // 1. 获取 ReentrantLock
    try{
        final Generation g = generation;

        if(g.broken){                       // 2. 判断 generation 是否已经 broken
            throw new BrokenBarrierException();
        }

        if(Thread.interrupted()){           // 3. 判断线程是否中断, 中断后就 breakBarrier
            breakBarrier();
            throw new InterruptedException();
        }

        int index = --count;                // 4. 更新已经到达 barrier 的线程数
        if(index == 0){ // triped           // 5. index == 0 说明所有线程到达了 barrier
            boolean ranAction = false;
            try{
                final Runnable command = barrierCommand;
                if(command != null){        // 6. 最后一个线程到达 barrier, 执行 command
                    command.run();
                }
                ranAction = true;
                nextGeneration();           // 7. 更新 generation
                return 0;
            }finally {
                if(!ranAction){
                    breakBarrier();
                }
            }
        }

        // loop until tripped, broken, interrupted, or timed out
        for(;;){
            try{
                if(!timed){
                    trip.await();           // 8. 没有进行 timeout 的 await
                }else if(nanos > 0L){
                    nanos = trip.awaitNanos(nanos); // 9. 进行 timeout 方式的等待
                }
            }catch (InterruptedException e){
                if(g == generation && !g.broken){ // 10. 等待的过程中线程被中断, 则直接唤醒所有等待的 线程, 重置 broken 的值
                    breakBarrier();
                    throw e;
                }else{
                    /**
                     * We're about to finish waiting even if we had not
                     * been interrupted, so this interrupt is deemed to
                     * "belong" to subsequent execution
                     */
                    /**
                     * 情况
                     *  1. await 抛 InterruptedException && g != generation
                     *      所有线程都到达 barrier(这是会更新 generation), 并且进行唤醒所有的线程; 但这时 当前线程被中断了
                     *      没关系, 当前线程还是能获取 lock, 但是为了让外面的程序知道自己被中断过, 所以自己中断一下
                     *  2. await 抛 InterruptedException && g == generation && g.broken = true
                     *      其他线程触发了 barrier broken, 导致 g.broken = true, 并且进行 signalALL(), 但就在这时
                     *      当前的线程也被 中断, 但是为了让外面的程序知道自己被中断过, 所以自己中断一下
                     *
                     */
                    Thread.currentThread().interrupt();
                }
            }



            if(g.broken){                       // 11. barrier broken 直接抛异常
                throw new BrokenBarrierException();
            }

            if(g != generation){                 // 12. 所有线程到达 barrier 直接返回
                return index;
            }

            if(timed && nanos <= 0L){           // 13. 等待超时直接抛异常, 重置 generation
                breakBarrier();
                throw new TimeoutException();
            }
        }
    }finally {
        lock.unlock();                          // 14. 调用 awaitXX 获取lock后进行释放lock
    }
}
```
### 其他方法
```java
/**
 * 判断 barrier 是否 broken = true
 */
public boolean isBroken(){
    final ReentrantLock lock = this.lock;
    lock.lock();
    try{
        return generation.broken;
    }finally {
        lock.unlock();
    }
}

// 重置 barrier
public void reset(){
    final ReentrantLock lock = this.lock;
    lock.lock();
    try{
        breakBarrier();  // break the current generation
        nextGeneration(); // start a new generation
    }finally {
        lock.unlock();
    }
}

/**
 * 获取等待中的线程
 */
public int getNumberWaiting(){
    final ReentrantLock lock = this.lock;
    lock.lock();
    try{
        return parties - count;
    }finally {
        lock.unlock();
    }
}
```