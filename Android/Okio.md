### Okio

okio的有点

* API简单易用.BufferedSink,BufferedSource提供了很多方便读写的函数读写行, 读写short,int,long等。用Okio和这个类很方便将InputStream,OutputStream转为BufferedSource，BufferedSink。ByteString这个类、用于不可变字节的操作非常方便提供了编解码加密等操作 与Buffer可变字节的间转换也很方便.

* Buffer内部用Segment的双向队列存储数据。Segment用data字节数组存数据,pos表示已读位置，limit表示

  data写入大小，shared表示data是不是被其他Segment共用。这样可以将读写放在一个buffer中进行,节省IO

  中临时数组的创建。Buffer的clone是通过共享的data的引用来实现，这样可以减少内存的开辟 

* 超时机制AsyncTimeout很方便使用 同过enter()加入优先级队列完成后要通过exit()移除,通过Watchdog监控队列超时后移除饼调用timeout()

---

基于例子分析

```java
BufferedSink sink2 = Okio.buffer(Okio.sink(file));
sink2.writeUtf8("Hello, java.io file!");
sink2.close();
```

第一步通过Okio.sink(file) 得到Okio中一个匿名内部sink1 该sink1持有file的FileOutputStream out

第二步Okio.buffer(sink1)得到一个RealBufferedSink对象sink2，sink2持有sink1及内部的一个Buffer对象buffer

第三步sink2.writeUtf8("Hello, java.io file!");首先用buffer.writeUtf8(string)写入到buffer中的一个Segment对象head的data中(head是一个Segment的双向队列)

第四步sink2.close();调用sink1.write(buffer) 将buffer中的数据写入out中 并关闭out

```java
BufferedSource source2 = Okio.buffer(Okio.source(file));
source2.readUtf8();
source2.close();
```

第一步通过Okio.source(file) 得到Okio中一个匿名内部source1 该source1持有file的FileInputStream in

第二步Okio.buffer(source1)得到一个RealBufferedSource对象source2，source2持有source1及内部的一个Buffer对象buffer

第三步source2.readUtf8();先调用buffer.writeAll(source1)将in中的内容写到buffer.head.data中

再调用 buffer.readUtf8();在readString(size, Util.UTF_8)中把buffer data数据转为一个字符串返回

第四步source2.close();关闭in流 清除buffer缓存



---

```java
Buffer data = new Buffer();
data.writeUtf8("a");
data.writeUtf8(repeat('b', 9998));
data.writeUtf8("c");

ByteArrayOutputStream out = new ByteArrayOutputStream();
Sink sink = Okio.sink(out);
sink.write(data, 3)
sink.write(data, data.size());
```
1. data.writeUtf8("a") 获取一个Segment对象head 在head的data中写入a，head的limit变为1
2. data.writeUtf8(repeat('b', 9998));head的limit没满8192 还是获取的head 在head的data中继续写入8192-1个字符。limit变为8192
3.  head满了重新获取一个segment做为head 写入1807个字符，limit->1807
4.  data.writeUtf8("c"); 在head中写入1个字符，limit->1808
5. Sink sink = Okio.sink(out); 得到Okio中一个匿名内部sink 该sink持有out
6. sink.write(data, 3) 获取head 在out中写入头3个字符 head.pos->3
7. sink.write(data, data.size()); 获取head 之前写了3个字符在out中写入8189个字符，head.pos->8192
8. 判断head.pos于limit相等 将head.next作为head  继续取出head.data写入out中。

---

```java
InputStream in = new ByteArrayInputStream(
    ("a" + repeat('b', Segment.SIZE * 2) + "c").getBytes(UTF_8));

// Source: ab...bc
Source source = Okio.source(in);
Buffer sink = new Buffer();

// Source: b...bc. Sink: abb.
assertEquals(3, source.read(sink, 3));
assertEquals("abb", sink.readUtf8(3));

// Source: b...bc. Sink: b...b.
assertEquals(Segment.SIZE, source.read(sink, 20000));
assertEquals(repeat('b', Segment.SIZE), sink.readUtf8());

```

1. 通过Okio.source(in) 得到Okio中一个匿名内部source 该source持有ByteArrayInputStream in

2. source.read(sink, 3)。sink获取一个Segment head,head.data中写入in前3个字符。head.limit->3返回3

3. sink.readUtf8(3)。得到head中前3个字符。head.pos->3  判断pos==limit  head->null

4. source.read(sink, 20000) ，重新获取一个head head.data中写入in往后8192个字符，limit->8192返回8192 size->8192

5. sink.readUtf8()  获取head中所有字符返回 head.pos->8192  判断pos==limit  head->null

---

