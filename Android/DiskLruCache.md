### DiskLruCache

---

阅读源码后还需要一篇好的[参考文章喽](https://blog.csdn.net/lmj623565791/article/details/47251585)

**自己的理解：**

DiskLruCache是一个最近最少使用磁盘上的存取方案，Lru还是基于LinkedHashMap实现

而存取暴露出来的都是流(存的时候拿到一个输出到文件的流，取得时候拿到一个从文件输入的流)

对 **journal**的理解,文件格式如下图。

![image-20181122221430960](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20181122221430960.png)

前面4行的参数用来判断是否需要重建一个journal。下面的内容用空格隔开分别代表操作类型，对应的key，最后面是文件的大小。类型有CLEAN：正常状态，DIRTY:用于删除dirty file. ,REMOVE: 删除状态,READ: 被读取状态.
readJournalLine方法会根据状态对lruEntries分别进行添加删除等操作，processJournal用于处理dirtyFile。通过以上处理 磁盘中的文件个数和大小的信息都被保存到了lruEntries中

**journal文件过大处理会变慢？**journalRebuildRequired方法用来判断。如果空闲条目大于2000且打印lruEntries的大小会去rebuildJournal.

**lruEntries中的Entry:**entry中有两个字段一个key一个long[] lengths。文件路径由key和数组下标表示

文件大小由数组中的值表示.

