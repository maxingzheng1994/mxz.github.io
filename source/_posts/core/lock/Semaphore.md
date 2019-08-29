# Semaphore
```java
        protected int tryAcquireShared(int acquires) {
            for (;;) {
                // 前置含有节点
                if (hasQueuedPredecessors())
                    return -1;
                // 更新state 当 available = 0 时;
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
```
声明 初始化state ， 限制 多少个并行 和 lock 是1