### ScheduledThreadPoolExecutor

 ScheduledThreadPoolExecutor继承自ThreadPoolExecutor.主要的功能是可以延迟和定期执行任务。

* ScheduledThreadPoolExecutor中用的是DelayedWorkQueue无限队列，所以最大线程数由corePoolSize决定
* ScheduledThreadPoolExecutor提交任务时不像ThreadPoolExecutor马上创建线程执行而是先提交到workQueue 然后开启线程 然后去workQueue拉取任务
* ScheduledThreadPoolExecutor的延时机制是在DelayedWorkQueue拉取任务时实现的。没达到时间拉取不到任务。
* DelayedWorkQueue中的数据结构是个二叉堆。因为task需要根据优先级排序。二叉堆添加删除时对结点排序比较快
*  scheduleAtFixedRate 和scheduleWithFixedDelay产生的区别原因是在setNextRunTime方法中。一个任务执行成功会调用setNextRunTime,scheduleAtFixedRate用task上一次的执行时间加上period，而scheduleWithFixedDelay是用当前时间加上period

---



```java
private void delayedExecute(RunnableScheduledFuture<?> task) {
    if (isShutdown())
        reject(task);
    else {
        //添加到workQueue 就是DelayedWorkQueue的offer方法
        super.getQueue().add(task);
        if (isShutdown() &&
            !canRunInCurrentRunState(task.isPeriodic()) &&
            remove(task))
            task.cancel(false);
        else
            ensurePrestart();
    }
}
```

```java
/**
 * Same as prestartCoreThread except arranges that at least one
 * thread is started even if corePoolSize is 0.
 */
void ensurePrestart() {
    int wc = workerCountOf(ctl.get());
    if (wc < corePoolSize)
        //现开启线程 再从workqueue中拉任务执行
        addWorker(null, true);
    else if (wc == 0)
        addWorker(null, false);
}
```

DelayedWorkQueue

```java
//将x加入二叉堆中
public boolean offer(Runnable x) {
    if (x == null)
        throw new NullPointerException();
    RunnableScheduledFuture<?> e = (RunnableScheduledFuture<?>)x;
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        int i = size;
        if (i >= queue.length)
            grow();
        size = i + 1;
        if (i == 0) {
            queue[0] = e;
            setIndex(e, 0);
        } else {
            siftUp(i, e);
        }
        if (queue[0] == e) {
            //有新任务提交了 唤醒等待成为leader的线程
            leader = null;
            available.signal();
        }
    } finally {
        lock.unlock();
    }
    return true;
}
```

```java
//会在ThreadPoolExecutor的getTask方法中调用。采用leader-follower模式
//这个任务拉取方法是实现延时的关键
public RunnableScheduledFuture<?> take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        for (;;) {
            RunnableScheduledFuture<?> first = queue[0];
            if (first == null)
                //没有任务等待
                available.await();
            else {
                //距离任务执行时间
                long delay = first.getDelay(NANOSECONDS);
                if (delay <= 0)
                    return finishPoll(first);
                first = null; // don't retain ref while waiting
                if (leader != null)//另外的task在执行
                    available.await();
                else {
                    Thread thisThread = Thread.currentThread();
                    //thisThread成为leader
                    leader = thisThread;
                    try {
                        available.awaitNanos(delay);
                    } finally {
                        //在available.awaitNanos(delay)过程中 其他线程是否成为了leader
                        //如果是 leader不能null
                        if (leader == thisThread)
                            leader = null;
                    }
                }
            }
        }
    } finally {
        if (leader == null && queue[0] != null)
            //如果leader为空 唤醒等待成为leader的线程
            available.signal();
        lock.unlock();
    }
}
```

---

```java
/**
 * Overrides FutureTask version so as to reset/requeue if periodic.
 */
public void run() {
    boolean periodic = isPeriodic();
    if (!canRunInCurrentRunState(periodic))
        cancel(false);
    else if (!periodic)
        ScheduledFutureTask.super.run();
    //这里是实现定时任务的关键 这判断是上个任务是否执行完成
    else if (ScheduledFutureTask.super.runAndReset()) {
        setNextRunTime();
        //outerTask就是原来的task 
        reExecutePeriodic(outerTask);
    }
}
```

```java
/**
 * Sets the next time to run for a periodic task.
 */
//scheduleAtFixedRate 和scheduleWithFixedDelay的区别在这 scheduleWithFixedDelay的period是负数。
private void setNextRunTime() {
    long p = period;
    if (p > 0)
        time += p;
    else
        time = triggerTime(-p);
}
```