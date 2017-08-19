CountDownLatch内部使用AQS实现的

首先通过构造函数初始化AQS的状态值


    public CountDownLatch(int count) {
        if (count < 0) throw new IllegalArgumentException("count < 0");
        this.sync = new Sync(count);
    }
        Sync(int count) {
            setState(count);
        }
然后看下await方法：

    public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        //如果线程被中断则抛异常
        if (Thread.interrupted())
            throw new InterruptedException();
        //尝试看当前是否计数值为0，为0则直接返回，否者进入队列等待
        if (tryAcquireShared(arg) < 0)
            doAcquireSharedInterruptibly(arg);
    }

 protected int tryAcquireShared(int acquires) {
            return (getState() == 0) ? 1 : -1;
        }
如果tryAcquireShared返回-1则 进入doAcquireSharedInterruptibly

    private void doAcquireSharedInterruptibly(int arg)
        throws InterruptedException {
       //加入队列状态为共享节点
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                       //如果多个线程调用了await被放入队列则一个个返回。
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        failed = false;
                        return;
                    }
                }
                //shouldParkAfterFailedAcquire会把当前节点状态变为SIGNAL类型，然后调用park方法把当先线程挂起，
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
调用await后，当前线程会被阻塞主，知道所有子线程调用了countdown方法，并在在计数为0时候调用该线程unpark方法激活线程，然后该线程重新tryAcquireShared会返回1。

然后看下 countDown方法：

委托给sync
    public void countDown() {
        sync.releaseShared(1);
    }
    public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
首先看下tryReleaseShared

        protected boolean tryReleaseShared(int releases) {
            //循环进行cas，直到当前线程成功完成cas使计数值（状态值state）减一更新到state
            for (;;) {
                int c = getState();
                if (c == 0)
                    return false;
                int nextc = c-1;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }

该函数一直返回false直到当前计数器为0时候才返回true。
返回true后会调用doReleaseShared，该函数主要作用是调用uppark方法激活调用await的线程，代码如下：

private void doReleaseShared() {

    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            //节点类型为SIGNAL，把类型在通过cas设置回去，然后调用unpark激活调用await的线程
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}
激活主线程后，主线程会在调用tryAcquireShared获取锁。