### netty

看了一天晕的。

先看一下java nio的ServeSocket实现 [Java NIO(6): Selector](https://zhuanlan.zhihu.com/p/27434028)

![image-20181204071325619](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20181204071325619.png)

```java
import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;
import java.util.Iterator;
import java.util.Set;

public class EpollServer {
    public static void main(String[] args) {
        try {
            ServerSocketChannel ssc = ServerSocketChannel.open();
            ssc.socket().bind(new InetSocketAddress("127.0.0.1", 8000));
            ssc.configureBlocking(false);

            Selector selector = Selector.open();
            // 注册 channel，并且指定感兴趣的事件是 Accept
            ssc.register(selector, SelectionKey.OP_ACCEPT);

            ByteBuffer readBuff = ByteBuffer.allocate(1024);
            ByteBuffer writeBuff = ByteBuffer.allocate(128);
            writeBuff.put("received".getBytes());
            writeBuff.flip();

            while (true) {
                int nReady = selector.select();
                Set<SelectionKey> keys = selector.selectedKeys();
                Iterator<SelectionKey> it = keys.iterator();

                while (it.hasNext()) {
                    SelectionKey key = it.next();
                    it.remove();

                    if (key.isAcceptable()) {
                        // 创建新的连接，并且把连接注册到selector上，而且，
                        // 声明这个channel只对读操作感兴趣。
                        SocketChannel socketChannel = ssc.accept();
                        socketChannel.configureBlocking(false);
                        socketChannel.register(selector, SelectionKey.OP_READ);
                    }
                    else if (key.isReadable()) {
                        SocketChannel socketChannel = (SocketChannel) key.channel();
                        readBuff.clear();
                        socketChannel.read(readBuff);

                        readBuff.flip();
                        System.out.println("received : " + new String(readBuff.array()));
                        key.interestOps(SelectionKey.OP_WRITE);
                    }
                    else if (key.isWritable()) {
                        writeBuff.rewind();
                        SocketChannel socketChannel = (SocketChannel) key.channel();
                        socketChannel.write(writeBuff);
                        key.interestOps(SelectionKey.OP_READ);
                    }
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

这个例子的关键点：

1. 创建一个ServerSocketChannel，和一个Selector，并且把这个server channel 注册到 selector上，注册的时间指定，这个channel 所感觉兴趣的事件是 SelectionKey.OP_ACCEPT，这个事件代表的是有客户端发起TCP连接请求。
2. 使用 select 方法阻塞住线程，当select 返回的时候，线程被唤醒。再通过selectedKeys方法得到所有可用channel的集合。
3. 遍历这个集合，如果其中channel 上有连接到达，就接受新的连接，然后把这个新的连接也注册到selector中去。
4. 如果有channel是读，那就把数据读出来，并且把它感兴趣的事件改成写。如果是写，就把数据写出去，并且把感兴趣的事件改成读。

---

带着这几个问题去看netty的HttpHelloWorldServer

1. 何时创建的ServerSocketChannel？ 

   在ServerBootstrap b调用bind时，doBind->initAndRegister->channelFactory().newChannel().

   channelFactory在b.channel(NioServerSocketChannel.class)时创建,factory的newChannel会去clazz.getConstructor().newInstance()构建一个NioServerSocketChannel最后通过newSocket，provider.openServerSocketChannel()创建一个ServerSocketChannel。用priveder去创建而不是用 ServerSocketChannel.open()去创建是因为open里也是去获取provider然后创建的。获取provider过程加了个锁，连接数大了会影响性能。

   在AbstractNioChannel的构造方法中调用ch.configureBlocking(false);

2. 何时创建的Selector？

   在EventLoopGroup bossGroup = new NioEventLoopGroup(1)的时候创建，group会去创建一个NioEventLoop。NioEventLoop是SingleThreadEventExecutor的一个子类。SingleThreadEventExecutor的构造方法中会创建一个thread。selector在NioEventLoop构造方法中创建.group().register(channel)调用时会去开启SingleThreadEventExecutor中的thread.

3. 何时将channel注册到selector上？

   SingleThreadEventLoop启动线程后 在会执行到AbstractNioChannel. doRegister()中执行

    selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);

4. 何时调用bind方法？

   会去pipeline中的tail开始调用bind方法 调用到AbstractChannelHandlerContext的bind方法会向前寻找outbound的ctx调用bind 最后会调用HeadContext去bind ,调用NioServerSocketChannel的doBind 最后调用

    javaChannel().bind(localAddress, config.getBacklog());

5. 何时及在哪个线程中调用select方法？

    NioEventLoop中的run在循环的调用 select ,int selectedKeys = selector.select(timeoutMillis);

   select获取事件后会在SelectedSelectionKeySet中添加一个事件，这是因为用反射将sun.nio.ch.SelectorImpl中的selectedKeys替换为了SelectedSelectionKeySet。

6. 何时去遍历selectedKeys的channel集合？

   也是在NioEventLoop的run方法中循环调用processSelectedKeys  判断SelectedSelectionKeySet是否有数据。

7. 何时去处理感兴趣的事件分别做如何处理？

---

 

 **NioEventLoop:**主要做I/O事件的一个监听及执行task

 **DefaultChannelPipeline：**  ChannelHandler的一个容器 其中有一个AbstractChannelHandlerContext双向列表 的责任链 inbound(被动的事件)事件会从head的handler开始向后处理.outbound(主动事件)会从tail的handler开始向前处理。