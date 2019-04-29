### Bucket

```java
final static int BITS_PER_WORD = Long.SIZE;//64位

final static long LAST_BIT = 1L << (Long.SIZE - 1);

long mData = 0;//

Bucket next;//超过64 列表的形式
```

整个类的作用就是操作一个long 上的0和1 支持插入删除 计数。 

[参考文章](<https://blog.csdn.net/u012227177/article/details/73381598>)