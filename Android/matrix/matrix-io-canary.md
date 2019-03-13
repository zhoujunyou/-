### matrix-io-canary

* **close leak检测机制**

  利用hook	dalvik.system.CloseGuard实现。

  CloseGuard的原理 那FileOutputStream分析

  * ![image-20181227172658761](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20181227172658761.png)

    这个guard就是CloseGuard对象 我们在创建一个FileOutputStream时调用open方法

  * ![image-20181227172754711](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20181227172754711.png)

    open方法中会判断是否开启 如果开启保留一个allocationSite.

  * ![image-20181227172900057](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20181227172900057.png)

    ![image-20181227173040354](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20181227173040354.png)



    FileOutputStream的close方法被调用 会去调用guard.close.会把allocationSite=null.

  * ![image-20181227173204986](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20181227173204986.png)

    ![image-20181227173229383](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20181227173229383.png)

    Gc触发后会调用FileOutputStream的finalize方法 如果allocationSize不为null表示未调用close 将堆栈信息报告出来。

#### Matrix  CloseGuardHooker 
hook机制

* ![image-20181227173828753](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20181227173828753.png)

  这里通过反射将一个动态代理设置到CloseGuard的REPORTER属性中。

* ![image-20181227174341872](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20181227174341872.png)

  hook后上面REPORTER调用reporter 后就会走到这里，我们在这里就可以处理自己的逻辑了。



####FileIOMainThreadDetector,FileIORepeatReadDetector,FileIOSmallBufferDetector检测机制

1. 框架是怎么知道我们调用了IO了呢？[Android中so文件的Hook](https://www.jianshu.com/p/dcb8f6b93ef9)  大概看明白了一点点。。。

   ![image-20181228140700264](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20181228140700264.png)



   ![image-20181228140618724](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20181228140618724.png)

   我们利用so so是ELF文件格式的hook机制将上面画线的函数替换成Proxy函数。

2. 

   ![image-20181228141046555](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20181228141046555.png)之后我们调用io open函数后会走到上面DoProxyOpenLogic函数中。kJavaBridgeClass对应java中的IOCanaryJniBridge,kMethodIDGetJavaContext对应IOCanaryJniBridge getJavaContext方法id，CallStaticObjectMethod调用方法。GetObjectField的到相应属性。

3. ![image-20181228145157403](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20181228145157403.png)

   info_map中插入fd,路径和JavaContext对应信息

   ![image-20181228160007221](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20181228160007221.png)

   ![image-20181228160216525](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20181228160216525.png)

   ![image-20181228160719684](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20181228160719684.png)

   每次调用read,write，close方法后会去重新计算相应IOInfo中上面的值，并向queue_中push数据。

4. ![image-20181228162013141](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20181228162013141.png)

   queue中接受到数据后 通知释放锁。进入detector.Detect detectors_中的数据就是在我们调用IOCanaryJniBridge.enableDetector方法时push进去的

5. ![image-20181228173413162](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20181228173413162.png)

   主线程耗时检测机制，kPossibleNegativeThreshold默认13ms,env.GetMainThreadThreshold()是在IOCanaryJniBridge.setConfig()方法传入我们自己配置的时间。

6. ![image-20181228183724898](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20181228183724898.png)

   根据path维护一个observing_map_。

   * 花费时间最大值小于13ms return
   * 有读操作清空repeat_infos 重置
   * 相比最近一次读取大于17ms 重置
   * 比较是不是同一个repeat_read_info 是同一个repeat_cnt_+1
   * 比较reapeat值是否大于我们设置的阈值

7. ![image-20181228185451904](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20181228185451904.png)

   文件读写操作大于20次。文件读写大小，读写次数比 小于我们设置的阈值判断为读写文件的buffer过小






