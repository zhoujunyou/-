### matrix-resource-canary-android

matrix-resource-canary-android负责检测内存泄漏 dump内存信息

1. Matrix build时会去调用ResourcePlugin的init方法得到ActivityRefWatcher。ResourcePlugin的start方法进入ActivityRefWatcher的start

2. ![image-20181226164308367](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20181226164308367.png)

   application注册一个ActivityLifecycleCallbacks回调 ，调用ActivityRefWatcher.scheduleDetectProcedure()

3. ![image-20181226164533042](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20181226164533042.png)

   判断该activity是否已经发布过了(在FilePublisher中用超时时间判断 默认24小时内判断过泄漏当作已发布)，对已判断为泄漏的Activity，记录其类名，避免重复提示该Activity已泄漏

   得到一个DestroyedActivityInfo 其中有String mKey，String mActivityName，WeakReference<Activity> mActivityRef，long mLastCreatedActivityCount，int mDetectedCount = 0有这几个属性。将DestroyedActivityInfo加到ConcurrentLinkedQueue<DestroyedActivityInfo> mDestroyedActivityInfos队列中

4. ![image-20181226165234577](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20181226165234577.png)

   mDetecteExectutor中包含了一个mainHandker和一个名为"default_matrix_thread"的HandlerThread 还有一个mDelayMillis控制轮训时间 这个时间通过IDynamicConfig中的clicfg_matrix_resource_detect_interval_millis获取 根据自己需要配置。IDynamicConfig中的get方法是一堆配置参数获取地。

5. ![image-20181226165926923](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20181226165926923.png)

   这段就是wiki中对应的 **Dalvik或ART虚拟机的GC机制我们是没法直接干预的，而且Android Framework也没有提供任何API让我们直接得知一个对象已被GC，但熟悉Java的同学大概很快就想到了WeakReference也许可以曲线救国我们可以通过创建一个持有已销毁的Activity的WeakReference，然后主动触发一次GC，如果这个Activity能被回收，则持有它的WeakReference会被置空，且这个被回收的Activity一段时间后会被加入事先注册到WeakReference的一个队列里。这样我们就能间接知道某个被销毁的Activity能否被回收了** 。 但是这里为啥是个Object? 如果调用gc后未回收等下一次轮训

6. ![image-20181226172550238](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20181226172550238.png)

   这里检测次数小于最大重检测次数或者destroy之后activity创建次数小于指定次数 等下一次检测 对应wiki描述

   - **VM并没有提供强制触发GC的API，通过`System.gc()`或`Runtime.getRuntime().gc()`只能“建议”系统进行GC，如果系统忽略了我们的GC请求，可回收的对象就不会被加入ReferenceQueue**
   - **将可回收对象加入ReferenceQueue需要等待一段时间，LeakCanary采用延时100ms的做法加以规避，但似乎并不绝对管用**

   - **监测逻辑是异步的，如果判断Activity是否可回收时某个Activity正好还被某个方法的局部变量持有，就会引起误判**
   - **若反复进入泄漏的Activity，LeakCanary会重复提示该Activity已泄漏**

   **对此我们做了以下改进：**

   - **增加一个一定能被回收的“哨兵”对象，用来确认系统确实进行了GC**

   - **直接通过`WeakReference.get()`来判断对象是否已被回收，避免因延迟导致误判**

   - **若发现某个Activity无法被回收，再重复判断3次，且要求从该Activity被记录起有2个以上的Activity被创建才认为是泄漏，以防在判断时该Activity被局部变量持有导致误判**

   - ****

7. ![image-20181226172844538](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20181226172844538.png)

   markPublished对已判断为泄漏的Activity，记录其类名，避免重复提示该Activity已泄漏,如果没有设置HeapDumper走轻量模式只报泄漏的activity否则去dump一个hprof文件参考LeakCanary

8.  CanaryWorkerService.shrinkHprofAndReport(context, result); 

    new HprofBufferShrinker().shrink(hprofFile, shrinkedHProfFile);

   ![image-20181226173428943](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20181226173428943.png)

   对应wiki 

   **裁剪Hprof**

   **Hprof文件的大小一般约为Dump时的内存占用大小，就微信而言Dump出来的Hprof大小通常为150MB～200MB之间，如果不做任何处理直接将此Hprof文件上传到服务端，一方面会消耗大量带宽资源，另一方面服务端将Hprof文件长期存档时也会占用服务器的存储空间。**

   **通过分析Hprof文件格式可知，Hprof文件中buffer区存放了所有对象的数据，包括字符串数据、所有的数组等，而我们的分析过程却只需要用到部分字符串数据和Bitmap的buffer数组，其余的buffer数据都可以直接剔除，这样处理之后的Hprof文件通常能比原始文件小1/10以上**。

9. ![image-20181226173632445](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20181226173632445.png)

   报告异常 将压缩后hprof文件路径及activity名字输出。



![image-20181226174409927](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20181226174409927.png)

 ResourcePlugin中的的一个方法用于修复InputMethodManager和Drawables导致报出的泄漏。



#### 与LeakCanary进行比较

这个模块参考了LeakCanary中的很多逻辑比较其不同点

1. 检测机制：LeakCanary在每次activity onDestroy后会进行一次gc检测 怀疑泄漏则dump内存继续分析是否泄漏。Matrix Canary在每次activity onDestroy后将其放入一个队列。会开一个线程轮循检测队列中对象是否泄漏。

2. 泄漏判断： 由于上面的机制不同。Matrix Canary必须在dump出hprof前准确判断是否泄漏所以进行了下列优化。
   - 增加一个一定能被回收的“哨兵”对象，用来确认系统确实进行了GC
   - 直接通过`WeakReference.get()`来判断对象是否已被回收，避免因延迟导致误判
   - 若发现某个Activity无法被回收，再重复判断3次，且要求从该Activity被记录起有2个以上的Activity被创建才认为是泄漏，以防在判断时该Activity被局部变量持有导致误判

3. 重复判断：LeakCanary 若反复进入泄漏的Activity，LeakCanary会重复提示该Activity已泄漏只有这个控制

   10分钟内dump过不进行dump。 

   Matrix Canary 对已判断为泄漏的Activity，记录其类名，避免重复提示该Activity已泄漏(默认记录24小时)

4. Hprof文件大小：LeakCanary完全是存储整个文件，而Matrix Canary对其进行了裁剪 官方描述：

   通过分析Hprof文件格式可知，Hprof文件中buffer区存放了所有对象的数据，包括字符串数据、所有的数组等，而我们的分析过程却只需要用到部分字符串数据和Bitmap的buffer数组，其余的buffer数据都可以直接剔除，这样处理之后的Hprof文件通常能比原始文件小1/10以上。

##### 对于上次整理的可以做以下优化点

![image-20181226180506622](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20181226180506622.png)	
1. Hprof裁剪。

2. InputMethodManager和Drawables导致误报的泄漏先进行过滤

3. 如果能忍受弱引用导致的延迟shi f 则维护一个泄漏队列。

4. 由于dump操作 Matrix Wiki描述如下**Dump Hprof阶段的开销则较大。Dump时整个App会卡死约5～15s，但考虑到ResourceCanary模块不会在线上环境启用，因此尚可接受。**

   所以还是只能让手动dump或打烊后上报（dump前保证gc成功）

