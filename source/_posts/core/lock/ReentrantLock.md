# ReentrantLock
通过构造方法创建两种类型的同步器 默认是不公平的
```java
    static final class NonfairSync extends Sync {
        final void lock() {
            if (compareAndSetState(0, 1)) // 加锁成功
            // 设置独占线程
                setExclusiveOwnerThread(Thread.currentThread());
            else
                // 否则加入等待队列
                acquire(1);
        }

        protected final boolean tryAcquire(int acquires)           {
            return nonfairTryAcquire(acquires);
        }
            // acquire 调用至此
       final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {   // 当前锁状态  再次加锁同上类似
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            // 是否是同一个线程  可重入锁
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            // 其他线程被锁住
            return false;
        }
    }
```

两者只有lock方 和 tryAcquire不一样
非公平锁的 lock 方法会先占锁再看状态
公平锁先hasQueuedPredecessors() 判断队列是否有等待的线程
```java
   static final class FairSync extends Sync {
        private static final long serialVersionUID = -3000897897090466540L;

        final void lock() {
            acquire(1); 
        }
        
        protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                //
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
    }
```
### tryLock
```java
    public final boolean tryAcquireNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        return tryAcquire(arg) ||   // 尝试获取锁 获取不到
            doAcquireNanos(arg, nanosTimeout);
    }
```
```java
 private boolean doAcquireNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (nanosTimeout <= 0L)
            return false;
        final long deadline = System.nanoTime() + nanosTimeout;  // 死亡时间
        final Node node = addWaiter(Node.EXCLUSIVE);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return true;
                }
                nanosTimeout = deadline - System.nanoTime();
                if (nanosTimeout <= 0L)
                    return false;
                if (11(p, node) &&
                    nanosTimeout > spinForTimeoutThreshold)
                    LockSupport.parkNanos(this, nanosTimeout);
                if (Thread.interrupted())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```
unLock




thread1： // cas 设置状态  exclusiveOwnerThread = thread1
thread2： // cas 设置状态失败 -> acquire -> tryAcquire(尝试获取锁，获取失败会加入队列)
            创建一个双向  New Node() <=>New Node(thread2, null) ,
            后面每个线程都加在队列后面， 并发加呢
            addWaiter  cas
thread3：               成功（加上去了）
thread4：                失败 （没加上去 去enq）遍历cas加入队列

加成功后 返回当前Node
    返回上一级Node 睡觉去

解锁
    更新状态直到状态减为0
    解锁成功 看头节点是否为空 且 等待状态不等于0
    唤醒头节点