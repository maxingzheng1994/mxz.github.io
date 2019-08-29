# ReentrantReadWriteLock

### writeLock
```java
protected final boolean tryAcquire(int acquires) {
            /*
             * Walkthrough:
             * 1. If read count nonzero or write count nonzero
             *    and owner is a different thread, fail.
             * 2. If count would saturate, fail. (This can only
             *    happen if count is already nonzero.)
             * 3. Otherwise, this thread is eligible for lock if
             *    it is either a reentrant acquire or
             *    queue policy allows it. If so, update state
             *    and set owner.
             */
            Thread current = Thread.currentThread();
            int c = getState();
            int w = exclusiveCount(c);  // 获取写锁(低16位)
            if (c != 0) {  // 当前有锁
                // (Note: if c != 0 and w == 0 then shared count != 0)
                // 写锁为0, 含有读锁 或者 持有锁的线程不是当前线程 返回 false 加Node到队列
                if (w == 0 || current != getExclusiveOwnerThread())
                    return false;
                //写入的锁  > 最大锁数量
                if (w + exclusiveCount(acquires) > MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                // 是当前线程 且拥有写锁 更新锁状态 获取成功
                setState(c + acquires);
                return true;
            }
            // 当前没有锁 // 是否有写锁  cas失败 说明被加锁了
            if (writerShouldBlock() ||
                !compareAndSetState(c, c + acquires))
                return false;
            setExclusiveOwnerThread(current);
            return true;
        }
```

### readLock
读锁 lock
```java
 protected final int tryAcquireShared(int unused) {

            Thread current = Thread.currentThread();
            int c = getState();
            // 含有写锁 且锁定线程不是当前线程
            if (exclusiveCount(c) != 0 &&
                getExclusiveOwnerThread() != current)
                return -1;
            int r = sharedCount(c); // 读锁
            // 只有读锁或当前线程是锁定线程
            // 读没被阻塞  且 加读锁成功
            if (!readerShouldBlock() &&
                r < MAX_COUNT &&
                compareAndSetState(c, c + SHARED_UNIT)) {
                if (r == 0) {
                    firstReader = current;
                    firstReaderHoldCount = 1;
                } else if (firstReader == current) {
                    firstReaderHoldCount++;
                } else {
                    HoldCounter rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current))
                        cachedHoldCounter = rh = readHolds.get();
                    else if (rh.count == 0)
                        readHolds.set(rh);
                    rh.count++;
                }
                return 1;
            }
            return fullTryAcquireShared(current);
        }
```
1. 先判断有没有写锁， 没有的话 或者当前线程不是加锁的线程
2.  给所继续加状态值。
3. 加读锁成功，
    1. 如果旧的读锁 为空 （当前线程为firstReader，count 1）
    2. 如果当前线程是firstReader count++
    3. 否则 判断cacheHolder 是否为null   将当前线程设置为缓存线程
    4. 如果当前线程的重入次数 = 0  将缓存放入threadLocal
    5. 重入次数加1
4. 加读锁失败 自旋
5. 含有写锁 却不是当前线程  会造成死锁
```java
 final int fullTryAcquireShared(Thread current) {

            HoldCounter rh = null;
            for (;;) {
                int c = getState(); 
                if (exclusiveCount(c) != 0) {
                    if (getExclusiveOwnerThread() != current)
                        return -1;
                    // else we hold the exclusive lock; blocking here
                    // would cause deadlock.
                } else if (readerShouldBlock()) {
                    // Make sure we're not acquiring read lock reentrantly
                    if (firstReader == current) {
                        // assert firstReaderHoldCount > 0;
                    } else {
                        if (rh == null) {
                            rh = cachedHoldCounter;
                            if (rh == null || rh.tid != getThreadId(current)) {
                                rh = readHolds.get();
                                if (rh.count == 0)
                                    readHolds.remove();
                            }
                        }
                        if (rh.count == 0)
                            return -1;
                    }
                }
                if (sharedCount(c) == MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                if (compareAndSetState(c, c + SHARED_UNIT)) {
                    if (sharedCount(c) == 0) {
                        firstReader = current;
                        firstReaderHoldCount = 1;
                    } else if (firstReader == current) {
                        firstReaderHoldCount++;
                    } else {
                        if (rh == null)
                            rh = cachedHoldCounter;
                        if (rh == null || rh.tid != getThreadId(current))
                            rh = readHolds.get();
                        else if (rh.count == 0)
                            readHolds.set(rh);
                        rh.count++;
                        cachedHoldCounter = rh; // cache for release
                    }
                    return 1;
                }
            }
        }
```