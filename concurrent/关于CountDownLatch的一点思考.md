> A synchronization aid that allows one or more threads to wait until a set of operations being performed in other threads completes.

**它允许一个或多个线程一直等待，直到其他线程的操作执行完后再执行。所以如果有一个操作需要在所以线程结束后执行，我们可以用它来进行处理。**  
例如下面，end是会在4个线程结束的时候才会打印
```
CountDownLatch latch = new CountDownLatch(4);
ExecutorService executorService = new ThreadPoolExecutor(4, 4,
        0L, TimeUnit.MILLISECONDS,
        new LinkedBlockingQueue<Runnable>());
System.out.println("start");
for (int i = 0; i < 4; i++) {
    executorService.submit(() -> {
                System.out.println("===task===");
                try {
                    Thread.sleep(2000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                latch.countDown();
            }
    );
}
try {
    latch.await();
} catch (InterruptedException e) {
    e.printStackTrace();
} finally{
    executorService.shutdown();
    System.out.println("===end===");
}
```
#### 为什么CountDownLatch可以实现？
CountDownLatch里面有个内部类Sync，Sync继承了AQS。AQS是一个同步队列器，他会将所有的线程添加进LCH，当state为0时，await操作才通过

```
public CountDownLatch(int count) {
    if (count < 0) throw new IllegalArgumentException("count < 0");
    this.sync = new Sync(count);
}

private static final class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = 4982264981922014374L;

    Sync(int count) {
        setState(count);
    }

    int getCount() {
        return getState();
    }

    protected int tryAcquireShared(int acquires) {
        return (getState() == 0) ? 1 : -1;
    }

    protected boolean tryReleaseShared(int releases) {
        // Decrement count; signal when transition to zero
        for (;;) {
            int c = getState();
            if (c == 0)
                return false;
            int nextc = c-1;
            if (compareAndSetState(c, nextc))
                return nextc == 0;
        }
    }
}

```
当一个线程处理结束调用latch.countDown();时，会调用Sync的releaseShared方法，使用了CAS(Compare And Swap)机制，来保证原子操作确保计数线程安全，并实现-1的效果，一直到state=0。CAS是通过JNI实现的，sun.misc.Unsafe 类提供了硬件级别的原子操作。但存在ABA的问题，需要注意(AtomicStampedReference 提供了根据版本号判断的实现)

```
// CountDownLatch
public void countDown() {
    sync.releaseShared(1);
}

// AQS
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}

/**
 * Release action for shared mode -- signals successor and ensures
 * propagation. (Note: For exclusive mode, release just amounts
 * to calling unparkSuccessor of head if it needs signal.)
 */
private void doReleaseShared() {
    /*
     * Ensure that a release propagates, even if there are other
     * in-progress acquires/releases.  This proceeds in the usual
     * way of trying to unparkSuccessor of head if it needs
     * signal. But if it does not, status is set to PROPAGATE to
     * ensure that upon release, propagation continues.
     * Additionally, we must loop in case a new node is added
     * while we are doing this. Also, unlike other uses of
     * unparkSuccessor, we need to know if CAS to reset status
     * fails, if so rechecking.
     */
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
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
// Sync
protected boolean tryReleaseShared(int releases) {
    // Decrement count; signal when transition to zero
    for (;;) {
        int c = getState();
        if (c == 0)
            return false;
        int nextc = c-1;
        if (compareAndSetState(c, nextc))
            return nextc == 0;
    }
}

// AQS JNI
protected final boolean compareAndSetState(int expect, int update) {
    // See below for intrinsics setup to support this
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```
当调用latch.await();时，会一直阻塞，等待结束
```
// CountDownLatch
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}
// AQS
public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}

/**
 * Acquires in shared interruptible mode.
 * @param arg the acquire argument
 */
private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```
