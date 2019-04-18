### HashMap



![image-20181101125606667](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20181101125606667.png)

HashMap的结构 数组+队列+红黑树。

一对k,v存入过程

1. 通过k的hash计算在数组中的位置table[i]  （table[]会在size >capital*loadFactor时加倍长度）即resize()
2. table[i]上数据少的时候是一个列表 多了就会转换为一个红黑树。
3. 在table[i]指向的列表或树中寻找相应k 有直接替换v 没有新建一个Node或TreeNode插到后面。

```java
static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```

(h = key.hashCode()) ^ (h >>> 16); 高位和低位的一个异或，以此来加大低位的随机性减少碰撞



```java
final Node<K,V>[] resize()
```

主要做Node<K,V>[] table的初始化，扩容以及扩容的时候的数据转移 