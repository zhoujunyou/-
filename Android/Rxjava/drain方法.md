###drain方法
一直不明白下面这些操作的作用 突然想了下

1. getAndIncrement() != 0 和decrementAndGet() == 0判断配合使用。第一次调用进入循环，

2. 如果decrementAndGet() == 0 不成立表示循环过程中有新的数据或状态加入继续循环 否则退出循环等下一次调用。

3. volatile 修饰的done用来标记状态是否结束  在其他线程修改done时保证了可见性 及时结束状态

4. 所以这里的queue.poll()不会存在多线程同时去拉情况 只有第一个调用循环的线程会去拉数据.

   保证了发送事件的顺序。

5. d && empty 判断用于如果有onNext 发生在complete之后 也先把数据发出来。就是

    Serializes calls to onNext, onError and onComplete.的实现。 

```java
volatile boolean done;
if (getAndIncrement() != 0) {
                return;
            }
for (;;) {
    boolean d = done;
    T v = queue.poll()

     if (d && empty) {
         e.onComplete();
         return;
       }    
    if (decrementAndGet() == 0) {
                    break;
                }
}

```

