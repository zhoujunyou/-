### AppendOnlyLinkedArrayList

开局一张图

![image-20181025225746561](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20181025225746561.png)

这是这个list的一个数据结构 每一块都是一个object  每一行都是object[] 每一行最后一列都有一个object[]引用

数据就是按照从左到右 从上到下的顺序添加

```java
/**
     * Constructs an empty list with a per-link capacity.
     * @param capacity the capacity of each link
     */
    public AppendOnlyLinkedArrayList(int capacity) {
        this.capacity = capacity;
        this.head = new Object[capacity + 1];
        this.tail = head;
    }

    /**
     * Append a non-null value to the list.
     * <p>Don't add null to the list!
     * @param value the value to append
     */
    public void add(T value) {
        final int c = capacity;
        int o = offset;
        //每次偏移量满足capacity 开辟一块object[]用tail[c]即最后一块去引用
        if (o == c) {
            Object[] next = new Object[c + 1];
            tail[c] = next;
            tail = next;
            o = 0;
        }
        tail[o] = value;
        offset = o + 1;
    }

/**
     * Loops over all elements of the array until a null element is encountered or
     * the given predicate returns true.
     * @param consumer the consumer of values that returns true if the forEach should terminate
     */
    @SuppressWarnings("unchecked")
    public void forEachWhile(NonThrowingPredicate<? super T> consumer) {
        Object[] a = head;
        final int c = capacity;
        while (a != null) {
            for (int i = 0; i < c; i++) {
                Object o = a[i];
                if (o == null) {
                    break;
                }
                if (consumer.test((T)o)) {
                    return;
                }
            }
            //当前object[]最后一项有数据 
            a = (Object[])a[c];
        }
    }
```



2. ### LinkedArrayList

   ![image-20181102155731723](/Users/meiweibuyondeng/Library/Application%20Support/typora-user-images/image-20181102155731723.png)

