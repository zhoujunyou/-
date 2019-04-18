

### Handler源码分析

1. Handler 提交消息到MessageQueue 以及处理消息
2. MessageQueue 存放消息队列
3. Looper 循环读取MessageQueue 交给Handler
4. Message 消息

#####何时创建Looper
主线程在ActivityThread main中调用了 Looper.prepareMainLooper();
将Looper对象存入ThreadLocal中 然后调用了Looper.loop()开启looper循环读取消息

其他线程则需要自己调用Looper.prepare()创建(ThreadLocal对象是每个线程独享的)。也会将Looper对象存入ThreadLocal中
因为获取Looper都通过sThreadLocal.get()获取所以Looper.prepare()需要在当前线程中调用
然后调用Looper.loop()开启消息读取。

```java
public static void prepare() {
        prepare(true);
    }
 private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
}
public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
    
public static @Nullable Looper myLooper() {
      return sThreadLocal.get();
}
```
所以在非主线程中创建Handler需要先调用Looper.prepare() 创建好Looper

#####何时创建MessageQueue

MessageQueue在Looper的构造方法中创建即调用Looper.prepare中创建。Handler每提交信息都会调用

MessageQueue 的enqueueMessage提交消息

```java
 private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }
```

##### 何时创建Message

无论在调用Handler send或者post方法时都会通过   Message.obtain()获取一个Message对象并在加入MessageQueue时将handler对象传递给Message的target用于后面分发消息。Message的对象都是从Message对象池里面拿的实例从而重复使用的,这也为了Android中的Message对象能够更好的回收


#####何时处理消息

前面Looper开启了loop循环 在MessageQueue中获取到提交的Message 调用Handler.dispatchMessage()

如果是post的消息则调用handleCallback(msg) 否则如果传入了Callback则调用Callback.handleMessage否则

调用Handler.handleMessage



![image-20180921150632919](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20180921150632919.png)盗用一张图文描述 形象生动[博客地址](https://blog.csdn.net/iispring/article/details/47180325)

我们可以把传送带上的货物看做是一个个的Message，而承载这些货物的传送带就是装载Message的消息队列MessageQueue。传送带是靠发送机滚轮带动起来转动的，我们可以把发送机滚轮看做是Looper，而发动机的转动是需要电源的，我们可以把电源看做是线程Thread，所有的消息循环的一切操作都是基于某个线程的。一切准备就绪，我们只需要按下电源开关发动机就会转动起来，这个开关就是Looper的loop方法，当我们按下开关的时候，我们就相当于执行了Looper的loop方法，此时Looper就会驱动着消息队列循环起来。
那Hanlder在传送带模型中相当于什么呢？我们可以将Handler看做是放入货物以及取走货物的管道：货物从一端顺着管道划入传送带，货物又从另一端顺着管道划出传送带。我们在传送带的一端放入货物的操作就相当于我们调用了Handler的sendMessageXXX、sendEmptyMessageXXX或postXXX方法，这就把Message对象放入到了消息队列MessageQueue中了。当货物从传送带的另一端顺着管道划出时，我们就相当于调用了Hanlder的dispatchMessage方法，在该方法中我们完成对Message的处理。



