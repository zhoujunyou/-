####ArrayList
ArrayLIst其实就是一个可变长度的数组Object[] elementData 封装了一些方法便于操作
每次add操作会去坚持数组是否够长 不够长用Arrays.copyOf方法加长数组
add(int index, E element)也是用System.arraycopy从index出开始拷贝到index+1 留出index位置 elementData[index]=element
get(index)直接返回elementData[index];
remove(index) 用System.arraycopy从index+1出开始拷贝到index  elementData[--size]=null;
remove(obj)遍历数组的到index 调用fastRemove(index);
iterator() 迭代也比较简单记录一个cursor用于next()  lastRet用于remove()

#### CopyOnWriteArrayList
线程安全的数组。相对于ArrayList 每次add的时候都会加个同步锁，并且每次都会新开辟一片数组。遍历的时候会持有数组的一个快照也就是老数组，所以在遍历过程中有添加数据也是获取不到的。








