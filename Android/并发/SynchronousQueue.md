# TIP

* SynchronousQueue

  源码比较绕 稍微过了下。先知道这个东西是干嘛的留个印象。

  是一个没有任何内部容量的队列.一个线程的插入必须等到有个线程拉取才能成功 反过来也一样。Executors.newFixedThreadPool 中使用的就是这个队列。 

  该线程池的适合用于大量且周期短的异步任务。可以复用之前存活的线程，如果没有就创建一个新的线程添加的池中，如果线程60秒没有任务终止。

  ```java
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
  ```

  因为corePoolSize为0 第一个判断失败。

  case1:第一个任务

  workQueue.offer(command) 时 workQueue没有poll操作 返回fasle

  addWorker(command, false)将会新建一个线程执行任务 再去getTask操作就会去workQueue.pool(60s)操作

  Case2:添加任务有线程正在pool

  workQueue.offer(command)成功 pool也立马成功开始执行任务 执行完 再getTask，pool。

  Case3:添加任务时没有在pool(可能超时，也可能都在执行任务没到pool)

  与case1逻辑一致

  ---


