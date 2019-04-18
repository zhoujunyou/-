### ArrayMap

**数据结构**

**一个int[] mHashes 有序数组存储key的hash**

 **一个Object[] mArray存储key和vaule。hash和key,value之间index关联。**

```java
mHashes[index] = hash;
mArray[index<<1] = key;  //等同于 mArray[index * 2] = key;
mArray[(index<<1)+1] = value; //等同于 mArray[index * 2 + 1] = value;
```

---

* **put**:先利用二分查找根据key在mHashes中查找index。binarySearch方法找到直接返回位置。未找到返回需要插入位置取~。

  判断返回如果负数则代表可以插入数据。负责替换原有value

  ```java
  static int binarySearch(int[] array, int size, int value) {
      int lo = 0;
      int hi = size - 1;
  
      while (lo <= hi) {
          int mid = (lo + hi) >>> 1;
          int midVal = array[mid];
  
          if (midVal < value) {
              lo = mid + 1;
          } else if (midVal > value) {
              hi = mid - 1;
          } else {
              return mid;  // value found
          }
      }
      return ~lo;  // value not present
  }
  ```

* **get**:也是通过二分法查找到下标 再用mArray[(index<<1)+1]得到value。



```java
/**
 * Caches of small array objects to avoid spamming garbage.  The cache
 * Object[] variable is a pointer to a linked list of array objects.
 * The first entry in the array is a pointer to the next array in the
 * list; the second entry is a pointer to the int[] hash code array for it.
 */
static Object[] mBaseCache;
static int mBaseCacheSize;
static Object[] mTwiceBaseCache;
static int mTwiceBaseCacheSize;
```

上面的这些变量用于缓存小数组 避免频繁GC.



---

##### 与hashmap相比较

优势：占用内存比较少。

劣势：数据多的时候hashmap查找效率大于arraymap(hashmap中key的hash值就能得到value位置，而arraymap需要先进行一次二分查找),arraymap插入,删除性能比hashmap要差主要用System.arraycopy实现。

