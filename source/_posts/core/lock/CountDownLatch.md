# CountDownLatch
```java
    CountDownLatch countDownLatch = new CountDownLatch(N);
        new Thread(() -> {
            System.out.println("kaishi");
            try {
                countDownLatch.await();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("jieshu");
        }).start();

        for (int i = 0; i < N; i++) {
            countDownLatch.countDown();
            Thread.sleep(500);
            System.out.println("11");
        }
```
```java
 1.       // 创建Sync
        Sync(int count) {
            setState(count);
        }   

2. 
    public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }
    public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (tryAcquireShared(arg) < 0)  // 当 state 小于0 时 等待失效
            doAcquireSharedInterruptibly(arg); 
    }
    
    private void doAcquireSharedInterruptibly(int arg)
        throws InterruptedException {
        final Node node = addWaiter(Node.SHARED); // 给链表再加一个等待者 cas 安全
        boolean failed = true;
        try {
            for (;;) {  // 遍历等待于此
                final Node p = node.predecessor(); // prev
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
    
    // 去掉一层state
    public void countDown() {
        sync.releaseShared(1);
    }
    
    public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {  // 遍历减1 直到为0 返回true
            doReleaseShared();
            return true;
        }
        return false;
    }
```