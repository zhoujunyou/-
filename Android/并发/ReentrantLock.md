



### ReentrantLock

` sun.misc.Unsafe`类的功能

下面方法可以获取一个对象的属性相对于该对象在内存当中的偏移量，这样我们就可以根据这个偏移量在对象内存当中找到这个属性

```java
long  stateOffset = unsafe.objectFieldOffset
                (AbstractQueuedSynchronizer.class.getDeclaredField("state"));

```

```java
/** 
* 比较obj的offset处内存位置中的值和期望的值，如果相同则更新。此更新是不可中断的。 
*  
* @param obj 需要更新的对象 
* @param offset obj中整型field的偏移量 
* @param expect 希望field中存在的值 
* @param update 如果期望值expect与field的当前值相同，设置filed的值为这个新值 
* @return 如果field的值被更改返回true 
*/  
public native boolean compareAndSwapInt(Object obj, long offset, int expect, int update); 
//和上面方法功能一样，只不过是设置Object类型的变量
public native boolean compareAndSwapObject(Object obj, long offset, Object expect, Object update);
```

```java
 //park函数是将当前调用Thread阻塞，而unpark函数则是将指定线程Thread唤醒。  
//park
    public native void park(boolean isAbsolute, long time);
    
    //unpack
    public native void unpark(Object var1);

```



非公平锁实现原理

1.  ReentrantLock.lock()

```java
public void lock() {
        sync.lock();
    }
```

2.  NonfairSync

```java
final void lock() {
    		//如果没有正在跑的线程 状态改为1
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }
```

3.  AbstractQueuedSynchronizer

```java
public final void acquire(int arg) {
    	//tryAcquire(arg) 为true 就会继续执行 否则进入acquireQueued方法
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```

4.  NonfairSync

```java
protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
```

5.  Sync

```java
final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
    		//获取当前状态
            int c = getState();
    		//表示没有线程在run
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
    		//同一个线程调用 重入锁
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```

6. AbstractQueuedSynchronizer

```java
private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
    	//如果tail指向的节点不为null 则向tail后添加结点
        if (pred != null) {
            node.prev = pred;
            //这里如果有保证多个线程同时走到这里 第一个会进入判断其他进入enq(node)
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;
    }
```

7. AbstractQueuedSynchronizer

```java
private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            //tail为null 新建一个Node 进入下一个循环
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                //将node 放到tail后返回node 
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
```

8. AbstractQueuedSynchronizer

```java
final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                //如果前一个结点就是head 表示当前node是第一个结点 
                if (p == head && tryAcquire(arg)) {
                    //相当于node被消耗了 将node作为head
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    //interrupted 这边会到步骤3。true中端当前线程 false则表示线程得到执行权限了
                    return interrupted;
                }
                //shouldParkAfterFailedAcquire 返回true即前一个节点状态是-1 执行parkAndCheckInterrupt()
//parkAndCheckInterrupt()会将当前线程阻塞 唤醒后在进入上面循环 因为此时新进来的线程有可能先进入tryAcquire 则这边需要继续park 这也是非公平锁的实现
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

9. AbstractQueuedSynchronizer

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    	//查看前一个结点状态 
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)
          	
            return true;
        if (ws > 0) {
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
           	//如果是0 waitStatus改为-1 等下一次循环
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
```

10.  AbstractQueuedSynchronizer

```java
private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }
```

11. LockSupport

```java
//使用 sun.misc.Unsafe中的方法 阻塞当前线程 
public static void park(Object blocker) {
        Thread t = Thread.currentThread();
        setBlocker(t, blocker);
        UNSAFE.park(false, 0L);
        setBlocker(t, null);
    }
```

12. ReentrantLock

```java
public void unlock() {
        sync.release(1);
    }
```

13. AbstractQueuedSynchronizer

```java
public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            //head的waitStatus不为0 
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```

14. Sync

```java
//该放方法将state-release 如果state为0 表示自由了
protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }
```

15. AbstractQueuedSynchronizer

```java
private void unparkSuccessor(Node node) {
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

        /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         */
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            //如果next为null 从tail回溯
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            //唤醒s节点中的线程 回到步骤10，9中
            LockSupport.unpark(s.thread);
    }
```



公平锁

16.  FairSync 

```java
//与2比较。每次都会去acquire   到3步骤
final void lock() {
            acquire(1);
        }
```

17.  FairSync

```java
 protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                //相比5 多了hasQueuedPredecessors判断队列中是否有等待的线程
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
```

18. AbstractQueuedSynchronizer

```java
//hasQueuedPredecessors中，如果tail和head不同，并且head的next为空或者head的next的线程不是当前线程，则表示队列不为空。有两种情况会导致h的next为空：
   //  1）当前线程进入hasQueuedPredecessors的同时，另一个线程已经更改了tail（在enq中），但还没有将head的next指向自己，这中情况表明队列不为空；
   //  2）当前线程将head赋予h后，head被另一个线程移出队列，导致h的next为空，这种情况说明锁已经被占用。
public final boolean hasQueuedPredecessors() {
        // The correctness of this depends on head being initialized
        // before tail and on head.next being accurate if the current
        // thread is first in queue.
        Node t = tail; // Read fields in reverse initialization order
        Node h = head;
        Node s;
        return h != t &&
            ((s = h.next) == null || s.thread != Thread.currentThread());
    }
```



### Condition 

[很不错的参考文章](https://www.jianshu.com/p/28387056eeb4)

19.  ConditionObject

```java
 public final void await() throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();
     		//将node追加到lastWaiter之后 ConditionObject中存在一个等待队列
            Node node = addConditionWaiter();
            int savedState = fullyRelease(node);
            int interruptMode = 0;
     		//不在AQS同步队列中  还没被阻塞
            while (!isOnSyncQueue(node)) {
                LockSupport.park(this);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }
```

20. ConditionObject

```java
 private Node addConditionWaiter() {
            Node t = lastWaiter;
            // If lastWaiter is cancelled, clean out.
            if (t != null && t.waitStatus != Node.CONDITION) {
                unlinkCancelledWaiters();
                t = lastWaiter;
            }
            Node node = new Node(Thread.currentThread(), Node.CONDITION);
            if (t == null)
                firstWaiter = node;
            else
                t.nextWaiter = node;
            lastWaiter = node;
            return node;
        }
```

21. ConditionObject

```java
 private void unlinkCancelledWaiters() {
            Node t = firstWaiter;
            Node trail = null;
     //用于清除队列中状态不是CONDITION的Node
            while (t != null) {
                Node next = t.nextWaiter;
                if (t.waitStatus != Node.CONDITION) {
                    t.nextWaiter = null;
                    if (trail == null)
                        firstWaiter = next;
                    else
                        trail.nextWaiter = next;
                    if (next == null)
                        lastWaiter = trail;
                }
                else
                    trail = t;
                t = next;
            }
        }
```

22. AbstractQueuedSynchronizer

```java
  final int fullyRelease(Node node) {
        boolean failed = true;
        try {
            int savedState = getState();
            //步骤13 释放同步队列中的线程 
            //释放的线程如果还走到await 继续来一遍到lastWaiter 那么只有signalAll能
            if (release(savedState)) {
                failed = false;
                return savedState;
            } else {
                throw new IllegalMonitorStateException();
            }
        } finally {
            if (failed)
                node.waitStatus = Node.CANCELLED;
        }
    }
```

23.  ConditionObject

```java
 public final void signal() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;
            if (first != null)
                doSignal(first);
        }
```

24. ConditionObject

```java
private void doSignal(Node first) {
            do {
                if ( (firstWaiter = first.nextWaiter) == null)
                    lastWaiter = null;
                first.nextWaiter = null;
            } while (!transferForSignal(first) &&
                     (first = firstWaiter) != null);
        }
```

25. AbstractQueuedSynchronizer

```java
final boolean transferForSignal(Node node) {
        /*
         * If cannot change waitStatus, the node has been cancelled.
         */
        if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
            return false;

        /*
         * Splice onto queue and try to set waitStatus of predecessor to
         * indicate that thread is (probably) waiting. If cancelled or
         * attempt to set waitStatus fails, wake up to resync (in which
         * case the waitStatus can be transiently and harmlessly wrong).
         */	
    	//步骤7 加到AQS的Sysn queue中  
        Node p = enq(node);
        int ws = p.waitStatus;
    	// await()之后的代码在lock.unlock()后执行。
        if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
            LockSupport.unpark(node.thread);
        return true;
    }
```

---

**lock 与 lockInterruptibly比较区别在于：**

​	**lock 优先考虑获取锁，待获取锁成功后，才响应中断。**

​	**lockInterruptibly 优先考虑响应中断，而不是响应锁的普通获取或重入获取。**

26. ReentrantLock

```java
public void lockInterruptibly() throws InterruptedException {
    sync.acquireInterruptibly(1);
}
```

27. AbstractQueuedSynchronizer

```java
public final void acquireInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (!tryAcquire(arg))
        doAcquireInterruptibly(arg);
}
```

28. AbstractQueuedSynchronizer

```java
private void doAcquireInterruptibly(int arg)
    throws InterruptedException {
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return;
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
```

