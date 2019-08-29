## AbstractQueuedSynchronizer
![](https://raw.githubusercontent.com/mxz1994/note/master/20190828210414.png)
state(共享资源)    FIFO线程等待队列

AQS定义两种资源共享方式：Exclusive（独占，只有一个线程能执行，如ReentrantLock）和Share（共享，多个线程可同时执行，如Semaphore/CountDownLatch）同时支持独占和共享（ReentrantReadWriteLock）
isHeldExclusively()：该线程是否正在独占资源。只有用到condition才需要去实现它。
tryAcquire(int)：独占方式。尝试获取资源，成功则返回true，失败则返回false。
tryRelease(int)：独占方式。尝试释放资源，成功则返回true，失败则返回false。
tryAcquireShared(int)：共享方式。尝试获取资源。负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源。
tryReleaseShared(int)：共享方式。尝试释放资源，如果释放后允许唤醒后续等待结点返回true，否则返回false。

以ReentrantLock为例，state初始化为0，表示未锁定状态。A线程lock()时，会调用tryAcquire()独占该锁并将state+1。此后，其他线程再tryAcquire()时就会失败，直到A线程unlock()到state=0（即释放锁）为止，其它线程才有机会获取该锁。当然，释放锁之前，A线程自己是可以重复获取此锁的（state会累加），这就是可重入的概念。但要注意，获取多少次就要释放多么次，这样才能保证state是能回到零态的

再以CountDownLatch以例，任务分为N个子线程去执行，state也初始化为N（注意N要与线程个数一致）。这N个子线程是并行执行的，每个子线程执行完后countDown()一次，state会CAS减1。等到所有子线程都执行完后(即state=0)，会unpark()主调用线程，然后主调用线程就会从await()函数返回，继续后余动作。

### 1.acquire
```java
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&                      //尝试直接去获取资源，如果成功则直接返回；
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))  // addWaiter()将该线程加入等待队列的尾部，并标记为独占模式；使线程在等待队列中获取资源，一直获取到资源后才返回。如果在整个等待过程中被中断过，则返回true，否则返回false。
            selfInterrupt(); //只是获取资源后才再进行自我中断selfInterrupt()，将中断补上
    }
```
#### addWaiter(Node)
```java
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);//mode = EXCLUSIVE(独占) SHARED(共享)
        Node pred = tail; // 初始值为null  
        if (pred != null) {  // 第二次进入node1不为null   加入队尾
            node.prev = pred;  //空Node <-> node1 <- node2
            if (compareAndSetTail(pred, node)) { //设置node2为队尾
                pred.next = node;    //空Node <-> node1 <-> node2
                return node;
            }
        }
        enq(node); //空Node <- node1   创建空节点头加到队尾
        return node;
    }
```
##### enq
```java
    private Node enq(final Node node) {
        //CAS"自旋"，直到成功加入队尾
        for (;;) {
            Node t = tail;  // 第二次循环空Node
            if (t == null) { // 头节点是null  加到头节点上
                if (compareAndSetHead(new Node()))
                    tail = head;  // 此时加入一个空Node
            } else {
                node.prev = t;  //空Node <- node1
                if (compareAndSetTail(t, node)) { // 更新tail
                    t.next = node;  //空Node <-> node1
                    return t; 
                }
            }
        }
    }
```
#### acquireQueued
```java
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            //cas自旋直到找到前置节点的状态为等待被唤醒状态(即找到休息位置)
            for (;;) {
                final Node p = node.predecessor();  
                if (p == head && tryAcquire(arg)) { // 当前节点的prev是头🐎， 是的话再次判断状态
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted; //返回等待过程中是否被中断过
                }
                 //如果自己可以休息了，就进入waiting状态，直到被unpark()
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())  // 在这里让出进入等待状态
                    interrupted = true;//如果等待过程中被中断过，哪怕只有那么一次，就将interrupted标记为true
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```
##### shouldParkAfterFailedAcquire
AQS在判断状态时，通过用waitStatus>0表示取消状态，而waitStatus<0表示有效状态。
```java
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;  // 前驱的状态
        if (ws == Node.SIGNAL) // 前驱为等待被唤醒
            return true;
        if (ws > 0) {
            // 前驱已被取消 再往前找上一个
            do { 
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node; 找到 有效的状态， 续上去
        } else {
            // 如果前驱正常 更新状态
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
    //如果前驱结点的状态不是SIGNAL，那么自己就不能安心去休息，需要去找个安心的休息点，同时可以再尝试下看有没有机会轮到自己拿号
```
##### parkAndCheckInterrupt
如果当前线程找好安全休息点,此方法就是让线程去休息，真正进入等待状态。
```java
    private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this); //调用park()使线程进入waiting状态
        return Thread.interrupted();//如果被唤醒，查看自己是不是被中断的。
    }
```
1. 结点进入队尾后，检查状态，找到安全休息点；
2. 调用park()进入waiting状态，等待unpark()或interrupt()唤醒自己；
3. 被唤醒后，看自己是不是有资格能拿到号。如果拿到，head指向当前结点，并返回从入队到拿到号的整个过程中是否被中断过；如果没拿到，继续流程1
![](_v_images/20190716174734916_17754.png)

### 2.release
```java
    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)  // 唤醒头节点后
                unparkSuccessor(h); //唤醒等待队列里的下一个线程
            return true;
        }
        return false;
    }
```
#### unparkSuccessor
```java
    private void unparkSuccessor(Node node) {
        int ws = node.waitStatus;
        if (ws < 0) // 有效状态更新头节点状态 = 0
            compareAndSetWaitStatus(node, ws, 0);

        Node s = node.next; /找到下一个需要唤醒的结点s
        if (s == null || s.waitStatus > 0) { // 下个节点是空的或者状态是无效的
            s = null;  // 清空下一个节点
            for (Node t = tail; t != null && t != node; t = t.prev) 
                if (t.waitStatus <= 0)
                    s = t; //遍历找到下一个有效状态的节点
        }
        if (s != null)  
            LockSupport.unpark(s.thread);//唤醒 
    }
```
　release()是独占模式下线程释放共享资源的顶层入口。它会释放指定量的资源，如果彻底释放了（即state=0）,它会唤醒等待队列里的其他线程来获取资源。
　
用unpark()唤醒等待队列中最前边的那个未放弃线程，这里我们也用s来表示吧。此时，再和acquireQueued()联系起来，s被唤醒后，进入if (p == head && tryAcquire(arg))的判断（即使p!=head也没关系，它会再进入shouldParkAfterFailedAcquire()寻找一个安全点。这里既然s已经是等待队列中最前边的那个未放弃线程了，那么通过shouldParkAfterFailedAcquire()的调整，s也必然会跑到head的next结点，下一次自旋p==head就成立啦），然后s把自己设置成head标杆结点，表示自己已经获取到资源了，acquire()也返回了

### 1. acquireShared(int)
```java
    public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }
```
#### doAcquireShared
```java
    private void doAcquireShared(int arg) {
        final Node node = addWaiter(Node.SHARED); // 加到队尾
        boolean failed = true;
        try {
            boolean interrupted = false;//等待过程中是否被中断过的标志
            for (;;) {
                final Node p = node.predecessor(); // 前驱
                if (p == head) { // 前驱 是头
                    int r = tryAcquireShared(arg); // 尝试获取资源
                    if (r >= 0) { // 成功
                        setHeadAndPropagate(node, r); //将head指向自己，还有剩余资源可以再唤醒之后的线程
                        p.next = null; // help GC
                        if (interrupted) //如果等待过程中被打断过，此时将中断补上。
                            selfInterrupt();
                        failed = false;
                        return;
                    }
                }
                //判断状态，寻找安全点，进入waiting状态，等着被unpark()或interrupt()
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```
##### setHeadAndPropagate
    private void setHeadAndPropagate(Node node, int propagate) {
        Node h = head; // Record old head for check below
        setHead(node); // 将当前Node 设为头
        //如果还有剩余量，继续唤醒下一个邻居线程
        if (propagate > 0 || h == null || h.waitStatus < 0 ||
            (h = head) == null || h.waitStatus < 0) {
            Node s = node.next;
            if (s == null || s.isShared())
                doReleaseShared();
        }
    }
tryAcquireShared()尝试获取资源，成功则直接返回；
失败则通过doAcquireShared()进入等待队列park()，直到被unpark()/interrupt()并成功获取到资源才返回。整个等待过程也是忽略中断的。
　　其实跟acquire()的流程大同小异，只不过多了个自己拿到资源后，还会去唤醒后继队友的操作（这才是共享）

### 2.releaseShared
```java
    public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
```
#### doReleaseShared
```java
 private void doReleaseShared() {
        for (;;) {
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                if (ws == Node.SIGNAL) {  //头节点为等待被唤醒
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0)) // 唤醒
                        continue;            // loop to recheck cases
                    unparkSuccessor(h); //唤醒后继
                }
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            if (h == head)                   // loop if head changed
                break;
        }
    }
```
## Node
```java
static final class Node {
        static final int CANCELLED =  1;
        static final int SIGNAL    = -1;
        static final int CONDITION = -2;
        static final int PROPAGATE = -3;
        volatile int waitStatus;
}
```
AQS在判断状态时，通过用waitStatus>0表示取消状态，而waitStatus<0表示有效状态。