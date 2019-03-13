### Reference

java中的引用类型做个简单记录

* **强引用** 垃圾回收器绝不会回收它，当内存空间不足，虚拟机宁愿抛出OutOfMemoryError错误，使程序异常终止，也不会靠回收具有强引用的对象来解决内存不足的问题。
*  **SoftReference(软引用)**  软引用的创建要借助于java.lang.ref包下的SoftReferenc类。当JVM进行垃圾回收时，只有在内存不足的时候JVM才会回收仅有软引用指向的对象所占的空间
* **WeakReference(弱引用) **弱引用的创建要借助于java.lang.ref包下的WeakReferenc类。当JVM进行垃圾回收时，无论内存是否充足，都会回收仅被弱引用关联的对象。由于垃圾回收器是一个优先级很低的线程，因此不一定会很快发现那些被弱引用指向的对象
* **PlantomReference(虚引用)**如果一个对象仅持有虚引用，那么它就和没有任何引用一样，在任何时候都可能被垃圾回收。创建一个虚引用对象时必须还要传递一个引用队列（ReferenceQueue）。

reference对应四种状态

1. Active:Active状态的Reference会受到GC的特别关注，当GC察觉到引用的可达性变化为其它的状态之后，它的状态将变化为Pending或Inactive，到底转化为Pending状态还是Inactive状态取决于此Reference对象创建时是否注册了queue.如果注册了queue，则将添加此实例到pending-Reference list中。 新创建的Reference实例的状态是Active
2. Pending:在pending-Reference list中等待着被Reference-handler 线程入队列queue中的元素就处于这个状态。没有注册queue的实例是永远不可能到达这一状态。
3. Enqueued:当实例被移动到ReferenceQueue中时，Reference的状态为Inactive。没有注册ReferenceQueue的不可能到达这一状态的
4. Inactive:一旦一个实例变为Inactive，则这个状态永远都不会再被改变。



#### ReferenceHandler

Reference的静态代码块中会创建一个最大优先级的ReferenceHandler守护线程不断的去遍历pending列表中是否有被gc回收的对象并将其enqueue到ReferenceQueue 没有则wait。pending是由JVM来赋值的





#### ReferenceQueue

Reference queues,在适当的时候检测到对象的可达性发生改变后，垃圾回收器就将已注册的引用对象添加到此队列中。

```java
boolean enqueue(Reference<? extends T> r) { /* Called only by Reference class */
    synchronized (lock) {
        // Check that since getting the lock this reference hasn't already been
        // enqueued (and even then removed)
        ReferenceQueue<?> queue = r.queue;
        if ((queue == NULL) || (queue == ENQUEUED)) {
            return false;
        }
        assert queue == this;
        r.queue = ENQUEUED;
        r.next = (head == null) ? r : head;
        head = r;
        queueLength++;
        if (r instanceof FinalReference) {
            sun.misc.VM.addFinalRefCount(1);
        }
        lock.notifyAll();
        return true;
    }
}

@SuppressWarnings("unchecked")
private Reference<? extends T> reallyPoll() {       /* Must hold lock */
    Reference<? extends T> r = head;
    if (r != null) {
        head = (r.next == r) ?
            null :
            r.next; // Unchecked due to the next field having a raw type in Reference
        r.queue = NULL;
        r.next = r;
        queueLength--;
        if (r instanceof FinalReference) {
            sun.misc.VM.addFinalRefCount(-1);
        }
        return r;
    }
    return null;
}
```



