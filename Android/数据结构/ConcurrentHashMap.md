### ConcurrentHashMap

数据结构： 数组+列表+红黑树

同步机制： synchronized(分段锁,数组中结点为锁对象)+CAS

扩容：         可以数组结点为单位多线程同时扩容



#### 1. 数据结构

 **Node类** 
Node是最核心的内部类，包装了key-value键值对，所有插入ConcurrentHashMap的数据都包装在这里面。 
它与HashMap中的定义很相似，但是有一些差别它对value和next属性设置了volatile同步锁，它不允许调用setValue方法直接改变Node的value域，它增加了find方法辅助map.get()方法。

**TreeNode类**
树节点类，另外一个核心的数据结构。 当链表长度过长的时候，会转换为TreeNode。 
但是与HashMap不相同的是，它并不是直接转换为红黑树，而是把这些结点包装成TreeNode放在TreeBin对象中，由TreeBin完成对红黑树的包装。 
而且TreeNode在ConcurrentHashMap继承自Node类，而并非HashMap中的集成自LinkedHashMap.Entry

**TreeBin** 
这个类并不负责包装用户的key、value信息，而是包装的很多TreeNode节点。它代替了TreeNode的根节点，也就是说在实际的ConcurrentHashMap“数组”中，存放的是TreeBin对象，而不是TreeNode对象，这是与HashMap的区别。他的构造方法传入一个头部TreeNode然后实现红黑树

**ForwardingNode **
一个用于连接两个table的节点类。它包含一个nextTable指针，用于指向下一张表。而且这个节点的key value next指针全部为null，它的hash值为-1. 这里面定义的find的方法是从nextTable里进行查询节点，而不是以自身为头节点进行查找。也是在多线程同步处理时的一个标记结点

----

####2. 同步机制

* **sizeCtl**

  1.负数代表正在进行初始化或扩容操作 
  2.-1代表正在初始化 
  3.-N 表示有N-1个线程正在进行扩容操作 
  4.正数或0代表hash表还没有被初始化，这个数值表示初始化或下一次进行扩容的大小，这一点类 	似于扩容阈值的概念。还后面可以看到，它的值始终是当前ConcurrentHashMap容量的0.7 倍，这与loadfactor是对应的。

* **tabAt,casTabAt,setTabAt**


```java
 static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
        return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
    }

    static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                        Node<K,V> c, Node<K,V> v) {
        return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
    }

    static final <K,V> void setTabAt(Node<K,V>[] tab, int i, Node<K,V> v) {
        U.putObjectVolatile(tab, ((long)i << ASHIFT) + ABASE, v);
    }
```

数组Node的操作全靠Unsafe类通过内存偏移量直接操作保证线程安全。

* **synchronized** 

  进行相应的添加,删除，扩容操作时会在table中找到相应的node  synchronized(node)后进行后续操作

  避免对整个map同步 只要table中node不同就可以进行并发操作

####3. 扩容

   **transferIndex** 

   这是扩容时	表示下一个需要被操作的table结点下标 当某个线程处理完分配的结点就会去申请新的任务这个下标会往前移

```java
 if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
            stride = MIN_TRANSFER_STRIDE; 
```

stride表示一个线程一次处理的结点数

* **单线程扩容** 

       新建两倍长的nextTable数组  table中结点从后i往前开始  
       
       普通的列表结点f  用fh&n值0和1 分为两个列表 ln,hn 分别加到nextTable的i,n+i处。
       
       TreeBin 同样分为两个列表 如果长度小于等于6 untreeify得到列表否则new TreeBin得到红黑树分别加到nextTable的i,n+i处。
       最后都会在table setTabAt(tab, i, fwd)处标记正在扩容.

* **多线程扩容**

  当线程对table结点进行操作发现是个(fh = f.hash) == MOVED 的ForwardingNode

  则会调用helpTransfer->transfer(tab, nextTab) 如果transferIndex>0帮助处理transferIndex-stride到transferIndex的结点 更新

  ```java
  U.compareAndSwapInt
                           (this, TRANSFERINDEX, nextIndex,
                            nextBound = (nextIndex > stride ?
                                         nextIndex - stride : 0))
  ```

  ****