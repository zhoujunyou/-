### ThreadPoolExecutor

[Java 线程池 ThreadPoolExecutor 源码分析](https://blog.csdn.net/cleverGump/article/details/50688008)

* ctl

  `private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));`

  ```
  // Packing and unpacking ctl
  private static int runStateOf(int c)     { return c & ~CAPACITY; }
  private static int workerCountOf(int c)  { return c & CAPACITY; }
  private static int ctlOf(int rs, int wc) { return rs | wc; }
  ```

   ctl是一个AtomicInteger  value高3位代表线程池状态低29位代表线程个数。runStateOf获取状态

  workerCountOf 获取线程数

* **rs线程池状态**

  ```java
  //能接受新提交的任务, 并且也能处理阻塞队列中的任务
  private static final int RUNNING    = -1 << COUNT_BITS;
  //不再接受新提交的任务, 但却可以继续处理阻塞队列中已保存的任务. 在线程池处于 RUNNING 状态时, 调用 //shutdown()方法会使线程池进入到该状态. 当然, finalize() 方法在执行过程中或许也会隐式地进入该状态.
  private static final int SHUTDOWN   =  0 << COUNT_BITS;
  //不能接受新提交的任务, 也不能处理阻塞队列中已保存的任务, 并且会中断正在处理中的任务. 在线程池处于 //RUNNING 或 SHUTDOWN 状态时, 调用 shutdownNow() 方法会使线程池进入到该状态
  private static final int STOP       =  1 << COUNT_BITS;
  // 所有的任务都已终止了, workerCount (有效线程数) 为0, 线程池进入该状态后会调用 terminated() 方法以让该线程池进入TERMINATED 状态. 当线程池处于 SHUTDOWN 状态时, 如果此后线程池内没有线程了并且阻塞队列内也没有待执行的任务了 (即: 二者都为空), 线程池就会进入到该状态. 当线程池处于 STOP 状态时, 如果此后线程池内没有线程了, 线程池就会进入到该状态.
  private static final int TIDYING    =  2 << COUNT_BITS;
  //terminated() 方法执行完后就进入该状态.
  private static final int TERMINATED =  3 << COUNT_BITS;
  ```

* **corePoolSize**,**maximumPoolSize**,**workQueue**

  ，workQueue阻塞队列

  1. corePoolSize核心线程数，表示的是线程池中一直存活着的线程的最小数量, 这些一直存活着的线程又被称为核心线程. 默认情况下, 核心线程的这个最小数量都是正数, 除非调用了allowCoreThreadTimeOut()方法并传递参数为true, 设置允许核心线程因超时而停止(terminated), 在那种情况下, 一旦所有的核心线程都先后因超时而停止了, 将使得线程池中的核心线程数量最终变为0, 也就是一直存活着的线程数为0, 这将是那种情况下, 线程池中核心线程数量的最小值. 默认情况下, 核心线程是按需创建并启动的, 也就是说, 只有当线程池接收到我们提交给他的任务后, 他才会去创建并启动一定数量的核心线程来执行这些任务. 如果他没有接收到相关任务, 他就不会主动去创建核心线程. 这种默认的核心线程的创建启动机制, 有助于降低系统资源的消耗. 变主动为被动, 类似于常见的观察者模式. 当然这只是系统默认的方式, 如果有特殊需求的话, 我们也可以通过调用 prestartCoreThread() 或 prestartAllCoreThreads() 方法来改变这一机制, 使得在新任务还未提交到线程池的时候, 线程池就已经创建并启动了一个或所有核心线程, 并让这些核心线程在池子里等待着新任务的到来.
  2. maximumPoolSize最大线程数,如果我们提供的阻塞队列 (也就是参数 workQueue) 是一个无界的队列, 那么这里提供的 maximumPoolSize 的数值将毫无意义, 这一点我们会在后面解释. 当我们通过方法 execute(Runnable) 提交一个任务到线程池时, 如果处于运行状态(RUNNING)的线程数量少于核心线程数(corePoolSize), 那么即使有一些非核心线程处于空闲等待状态, 系统也会倾向于创建一个新的线程来处理这个任务. 如果此时处于运行状态(RUNNING)的线程数量大于核心线程数(corePoolSize), 但又小于最大线程数(maximumPoolSize), 那么系统将会去判断线程池内部的阻塞队列 workQueue 中是否还有空位子. 如果发现有空位子, 系统就会将该任务先存入该阻塞队列; 如果发现队列中已没有空位子(即: 队列已满), 系统就会新创建一个线程来执行该任务.如果将线程池的核心线程数 corePoolSize 和 最大线程数 maximumPoolSize 设置为相同的数值(也就是说, 线程池中的所有线程都是核心线程), 那么该线程池就是一个容量固定的线程池. 如果将最大线程数 maximumPoolSize 设置为一个非常大的数值(例如: Integer.MAX_VALUE), 那么就相当于允许线程池自己在不同时段去动态调整参与并发的任务总数. 通常情况下, 核心线程数 corePoolSize 和 最大线程数 maximumPoolSize 仅在创建线程池的时候才去进行设定, 但是, 如果在线程池创建完成以后, 你又想去修改这两个字段的值, 你就可以调用 setCorePoolSize() 和 setMaximumPoolSize() 方法来分别重新设定核心线程数 corePoolSize 和 最大线程数 maximumPoolSize 的数值.
  3. workQueue 部元素为 Runnable(各种任务, 通常是异步的任务) 的阻塞队列. 阻塞队列是一种类似于 “生产者 - 消费者”模型的队列. 当队列已满时如果继续向队列中插入元素, 该插入操作将被阻塞一直处于等待状态, 直到队列中有元素被移除产生空位子后, 才有可能执行这次插入操作; 当队列为空时如果继续执行元素的删除或获取操作, 该操作同样会被阻塞而进入等待状态, 直到队列中又有了该元素后, 才有可能执行该操作.

  ---

  **(1) 如果线程池中正在运行的线程数少于核心线程数, 那么线程池总是倾向于创建一个新线程来执行该任务, 而不是将该任务提交到该队列 workQueue 中进行等待.** 
  **(2) 如果线程池中正在运行的线程数不少于核心线程数, 那么线程池总是倾向于将该任务先提交到队列 workQueue 中先让其等待, 而不是创建一个新线程来执行该任务.** 
  **(3) 如果线程池中正在运行的线程数不少于核心线程数, 并且线程池中的阻塞队列也满了使得该任务入队失败, 那么线程池会去判断当前池子中运行的线程数是否已经等于了该线程池允许运行的最大线程数 maximumPoolSize. 如果发现已经等于了, 说明池子已满, 无法再继续创建新的线程了, 那么就会拒绝执行该任务. 如果发现运行的线程数小于池子允许的最大线程数, 那么就会创建一个线程(这里创建的线程是非核心线程)来执行该任务.**

  ---

  ```java
   public void execute(Runnable command) {
          if (command == null)
              throw new NullPointerException();
          /*
           * Proceed in 3 steps:
           *
           * 1. If fewer than corePoolSize threads are running, try to
           * start a new thread with the given command as its first
           * task.  The call to addWorker atomically checks runState and
           * workerCount, and so prevents false alarms that would add
           * threads when it shouldn't, by returning false.
           *
           * 2. If a task can be successfully queued, then we still need
           * to double-check whether we should have added a thread
           * (because existing ones died since last checking) or that
           * the pool shut down since entry into this method. So we
           * recheck state and if necessary roll back the enqueuing if
           * stopped, or start a new thread if there are none.
           *
           * 3. If we cannot queue task, then we try to add a new
           * thread.  If it fails, we know we are shut down or saturated
           * and so reject the task.
           */
          int c = ctl.get();
          if (workerCountOf(c) < corePoolSize) {
              if (addWorker(command, true))
                  return;
              c = ctl.get();
          }
          if (isRunning(c) && workQueue.offer(command)) {
              int recheck = ctl.get();
              if (! isRunning(recheck) && remove(command))
                  reject(command);
              else if (workerCountOf(recheck) == 0)
                  addWorker(null, false);
          }
          else if (!addWorker(command, false))
              reject(command);
      }
  ```

  1.  如果线程池内的有效线程数少于核心线程数 corePoolSize, 那么就创建并启动一个线程来执行新提交的任务. 
  2.  如果线程池内的有效线程数达到了核心线程数 corePoolSize, 并且线程池内的阻塞队列未满, 那么就将新提交的任务加入到该阻塞队列中.
  3.  如果线程池内的有效线程数达到了核心线程数 corePoolSize 但却小于最大线程数 maximumPoolSize, 并且线程池内的阻塞队列已满, 那么就创建并启动一个线程来执行新提交的任务. 
  4.  如果线程池内的有效线程数达到了最大线程数 maximumPoolSize, 并且线程池内的阻塞队列已满, 那么就让 RejectedExecutionHandler 根据它的拒绝策略来处理该任务, 默认的处理方式是直接抛异常.

![image-20181115105710275](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20181115105710275.png)

```java
 /**
     * Checks if a new worker can be added with respect to current
     * pool state and the given bound (either core or maximum). If so,
     * the worker count is adjusted accordingly, and, if possible, a
     * new worker is created and started, running firstTask as its
     * first task. This method returns false if the pool is stopped or
     * eligible to shut down. It also returns false if the thread
     * factory fails to create a thread when asked.  If the thread
     * creation fails, either due to the thread factory returning
     * null, or due to an exception (typically OutOfMemoryError in
     * Thread.start()), we roll back cleanly.
     *
     * @param firstTask the task the new thread should run first (or
     * null if none). Workers are created with an initial first task
     * (in method execute()) to bypass queuing when there are fewer
     * than corePoolSize threads (in which case we always start one),
     * or when the queue is full (in which case we must bypass queue).
     * Initially idle threads are usually created via
     * prestartCoreThread or to replace other dying workers.
     *
     * @param core if true use corePoolSize as bound, else
     * maximumPoolSize. (A boolean indicator is used here rather than a
     * value to ensure reads of fresh values after checking other pool
     * state).
     * @return true if successful
     */
private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c);
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int rs = runStateOf(ctl.get());

                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
```



```java
/**
     * Main worker run loop.  Repeatedly gets tasks from queue and
     * executes them, while coping with a number of issues:
     *
     * 1. We may start out with an initial task, in which case we
     * don't need to get the first one. Otherwise, as long as pool is
     * running, we get tasks from getTask. If it returns null then the
     * worker exits due to changed pool state or configuration
     * parameters.  Other exits result from exception throws in
     * external code, in which case completedAbruptly holds, which
     * usually leads processWorkerExit to replace this thread.
     *
     * 2. Before running any task, the lock is acquired to prevent
     * other pool interrupts while the task is executing, and then we
     * ensure that unless pool is stopping, this thread does not have
     * its interrupt set.
     *
     * 3. Each task run is preceded by a call to beforeExecute, which
     * might throw an exception, in which case we cause thread to die
     * (breaking loop with completedAbruptly true) without processing
     * the task.
     *
     * 4. Assuming beforeExecute completes normally, we run the task,
     * gathering any of its thrown exceptions to send to afterExecute.
     * We separately handle RuntimeException, Error (both of which the
     * specs guarantee that we trap) and arbitrary Throwables.
     * Because we cannot rethrow Throwables within Runnable.run, we
     * wrap them within Errors on the way out (to the thread's
     * UncaughtExceptionHandler).  Any thrown exception also
     * conservatively causes thread to die.
     *
     * 5. After task.run completes, we call afterExecute, which may
     * also throw an exception, which will also cause thread to
     * die. According to JLS Sec 14.20, this exception is the one that
     * will be in effect even if task.run throws.
     *
     * The net effect of the exception mechanics is that afterExecute
     * and the thread's UncaughtExceptionHandler have as accurate
     * information as we can provide about any problems encountered by
     * user code.
     *
     * @param w the worker
     */
final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            while (task != null || (task = getTask()) != null) {
                w.lock();
                // If pool is stopping, ensure thread is interrupted;
                // if not, ensure thread is not interrupted.  This
                // requires a recheck in second case to deal with
                // shutdownNow race while clearing interrupt
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);
        }
    }
```

```java
 /**
     * Performs blocking or timed wait for a task, depending on
     * current configuration settings, or returns null if this worker
     * must exit because of any of:
     * 1. There are more than maximumPoolSize workers (due to
     *    a call to setMaximumPoolSize).
     * 2. The pool is stopped.
     * 3. The pool is shutdown and the queue is empty.
     * 4. This worker timed out waiting for a task, and timed-out
     *    workers are subject to termination (that is,
     *    {@code allowCoreThreadTimeOut || workerCount > corePoolSize})
     *    both before and after the timed wait, and if the queue is
     *    non-empty, this worker is not the last thread in the pool.
     *
     * @return task, or null if the worker must exit, in which case
     *         workerCount is decremented
     */
    private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?

        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                decrementWorkerCount();
                return null;
            }

            int wc = workerCountOf(c);

            // Are workers subject to culling?
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }

            try {
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }
```

---
 **shutdown()**方法将线程池改为SHUTDOWN状态 将无法提交任务, 下面时候getTask中代码如果workqueue为空结束线程 否则继续执行 所以shutdown后队列中如果有任务回继续执行完

 **shutdownNow()线程池改为STOP状态** workqueue中任务也不会执行了。然后返回workqueue中的任务列表.

```java
if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                decrementWorkerCount();
                return null;
            }
```

