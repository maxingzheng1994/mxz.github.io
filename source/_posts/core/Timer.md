# Timer
Timer 工具类
schedule(TimerTask task, Date time)：安排在指定的时间点执行指定的任务
schedule(TimerTask task, long delay) ：安排在指定延迟后执行指定的任务。
schedule(TimerTask task, long delay, long period) ：安排指定的任务从指定的延迟后开始进行重复的固定延迟执行  -> sched(task, System.currentTimeMillis()+delay, -period);  
                            ->  currentTime   - task.period   当前执行时间的下一个循环周期
scheduleAtFixedRate(TimerTask task, long delay, long period) 区别 period 
    -> sched(task, System.currentTimeMillis()+delay, period);
        -> executionTime + task.period  下次执行时间再过一个循环周期
```java 
// 计算下次执行时间  
    queue.rescheduleMin(
      task.period<0 ? currentTime   - task.period
                    : executionTime + task.period);  
}
```


程序运行：
在初始化Timer时 ，开启一个线程循环提取任务数组中的任务，如果任务数组为空，线程等待直到添加任务；当添加任务时，唤醒线程，提取数组中标记为1的任务，如果该任务状态为CANCELLED，则从数组中删除任务，continue ，继续循环提取任务；然后将当前时间与任务执行时间点比较 标记taskFired=executionTime<=currentTime; taskFired =false ，说明任务执行时间还没到，则调用wait等待(executionTime-currentTime) 时间长度，然后循环重新提取该任务；
taskFired =true，说明任务执行时间已经到了，或者过去了。继续判断 任务循环时间间隔period；
 period=0时，说明此次任务是非循环任务，直接将该任务从数组中删除，并将状态置为EXECUTED，然后执行任务的run方法 period！=0时，说明此次任务时循环任务，将该任务的执行时间点向前推进，具体推进时间根据调用的方法判断；如果是schedule方法，则在当前时间基础上向前推进period时间长度； 如果是scheduleAtFixedRate方法，则在当前任务执行时间点基础上向前推进period时间长度，
 最后执行任务的run方法；循环提取任务

```java
// 启动一个线程定期从TimeTask 中取任务
  public Timer(String name) {
        thread.setName(name);
        thread.start();
    }
```
TaskThread
```java
    public void run() {
        try {
            mainLoop();
        } finally {
            // Someone killed this Thread, behave as if Timer cancelled
            synchronized(queue) {
                newTasksMayBeScheduled = false;
                queue.clear();  // Eliminate obsolete references
            }
        }
    }
    
     private void mainLoop() {
        while (true) {
            try {
                TimerTask task;
                boolean taskFired;
                synchronized(queue) {
                    // Wait for queue to become non-empty
                    while (queue.isEmpty() && newTasksMayBeScheduled)
                        queue.wait();
                    if (queue.isEmpty())
                        break; // Queue is empty and will forever remain; die

                    // Queue nonempty; look at first evt and do the right thing
                    long currentTime, executionTime;
                    task = queue.getMin(); // queue[1]
                    synchronized(task.lock) {
                        if (task.state == TimerTask.CANCELLED) {
                            queue.removeMin(); // 任务已取消移除
                            continue;  // No action required, poll queue again
                        }
                        currentTime = System.currentTimeMillis();
                        executionTime = task.nextExecutionTime;
                        if (taskFired = (executionTime<=currentTime)) {
                            if (task.period == 0) { // Non-repeating, remove
                                queue.removeMin();
                                task.state = TimerTask.EXECUTED;
                            } else { // Repeating task, reschedule
                                queue.rescheduleMin(
                                  task.period<0 ? currentTime   - task.period
                                                : executionTime + task.period);
                            }
                        }
                    }
                    if (!taskFired) // Task hasn't yet fired; wait
                        queue.wait(executionTime - currentTime);
                }
                if (taskFired)  // Task fired; run it, holding no locks
                    task.run();
            } catch(InterruptedException e) {
            }
        }
    }
}
```

使用Timer 当其中某个任务执行时间过长， 会影响其他任务
或者当其中一个任务抛出异常后会影响其他任务

用 ScheduledExecutorService  代替Timer

对于Timer的缺陷，我们可以考虑 ScheduledThreadPoolExecutor 来替代。Timer是基于绝对时间的，对系统时间比较敏感，而ScheduledThreadPoolExecutor 则是基于相对时间；Timer是内部是单一线程，而ScheduledThreadPoolExecutor内部是个线程池，所以可以支持多个任务并发执行。

