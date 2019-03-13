### LruCache

Least Recently Used 最近最少使用算法

最近最少使用排序内部主要是用LinkedHashMap完成

思想就是限定一个size，put时加上value的size 判断是否超过 超过就将最少使用的移除
remove时减去value的size。最后由于LinkedHashMap不是线程安全的 操作时都需要加上synchronized

