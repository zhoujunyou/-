### synchronized

[参考文章](https://www.jianshu.com/p/ea9a482ece5f)

1. synchronized方法 同一个对象中的synchronized方法会互斥 因为当前对象是锁

2. 静态synchronized方法会与类中synchronized方法互斥 因为类Class对象是锁。

3. synchronized代码块 可以用byte[]对象当锁

   注：零长度的byte数组对象创建起来将比任何对象都经济――查看编译后的字节码：生成零长度的byte[]对象只需3条操作码，而Object lock = new Object()则需要7行操作码



4. synchronized 是一个可重入锁。重入的定义：当某个线程请求一个由其他线程持有的锁时，发出请求的线程就会阻塞。然而由于内置锁是可重入的，因此如果某个线程试图获得一个已经由他自己持有的锁，那么这个请求就会成功。“重入” 意味着获取锁的操作粒度是“线程” 而不是调用

