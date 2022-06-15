### 1. 概览

**AbstractQueuedSynchronizer**是juc包下提供的锁，可以用来解决线程安全问题。



### 2. 锁的猜想

### 3.源码分析

下面以非公平锁的源码进行分析，其实公平锁和非公平锁的区别就在于，非公平锁不会放过任何一次可以抢站锁的机会，而公平锁是在没有抢占锁的情况下会进入到等待队列中，等待其他线程释放锁后进行唤醒。



##### 1. lock()

```java
public void lock() {
    //以非公平锁为例，调用NonfairSync内部类中的Lock方法
    sync.lock();
}

final void lock() {
    //这里体现了非公平锁的区别，先通过cas操作设置state的值为1，如果设置成功，代表获取锁成功，修改当前线程为独占状态（如果是公平锁的话，没有这个步骤，直接调用acquire方法）
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
    else
        //如果获取锁失败，执行这里，重点关注
        acquire(1);
}
```



##### 2.acquire()

```java
public final void acquire(int arg) {
    //该方法主要有三个核心的方法分别是：
    //1.tryAcquire(arg) 再次去获取锁，以及可重入锁的实现
    //2.addWaiter(Node.EXCLUSIVE) 把当前节点
    //3.acquireQueued(addWaiter(Node.EXCLUSIVE), arg)
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

- nonfairTryAcquire()

```java
final boolean nonfairTryAcquire(int acquires) {
    //获取当前线程
    final Thread current = Thread.currentThread();
    //获取state变量
    int c = getState();
    //如果等于零代表还没有线程抢占锁
    if (c == 0) {
        //通过cas操作修改state状态变量，修改成果代表获取锁成功，设置当前线程为独占状态，返回true
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    //如果state变量不为零，判断是不是同一个线程获取锁，也就是可重入锁的逻辑，并且不需要进行加锁操作，因为都是同一个线程
    else if (current == getExclusiveOwnerThread()) {
        //state变量+1
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    //以上条件都不满足，则代表当前线程抢占锁失败，返回false
    return false;
}
```

- addWaiter()

调用`nonfairTryAcquire`方法获取锁失败后，会调用该方法，把当前线程封装成Node，放入到等待队列中

```java
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}

private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```

- acquireQueued

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}


private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
        return true;
    if (ws > 0) {
        /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```

