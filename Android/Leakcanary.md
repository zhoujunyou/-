### Leakcanary

 * System.gc(); //告诉垃圾收集器打算进行垃圾收集，而垃圾收集器进不进行收集是不确定的
 * System.runFinalization(); //强制调用已经失去引用的对象的finalize方法 
 *  Debug.isDebuggerConnected()判断apk是否处于调式状态

##### 检测activity, fragment泄漏的原理

 ActivityRefWatcher

```
application.registerActivityLifecycleCallbacks(activityRefWatcher.lifecycleCallbacks);
```

 SupportFragmentRefWatcher

```
supportFragmentManager.registerFragmentLifecycleCallbacks(fragmentLifecycleCallbacks, true);
```

在activity onDestroy,fragment onDestroy时会去调用RefWatcher的watch

将activity或fragment用KeyedWeakReference引用并将其放入ReferenceQueue，KeyedWeakReference是个弱引用。观察对象destroy有没有加入ReferenceQueue，如果有表示被回收。如果没有，手动gc后再次查看是否放入了queue中 还是没有表示对象泄漏了 调用heapDumper.dumpHeap()得到hprof文件

![image-20181225172330760](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20181225172330760.png)

利用haha库解析hprof文件得到一个Snapshot对象。

![image-20181225172740693](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20181225172740693.png)

1. 先找到KeyedWeakReference class 对象
2. 找到KeyedWeakReference对象实例子
3. 找到对象中key字段
4. 比较key字段判断是不是我们监控的对象的KeyedWeakReference
5. 返回我们监控的对象referent

![image-20181225173726094](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20181225173726094.png)

1. 获取gc root到监控对象的最短引用路径。
2. 不存在表示不泄漏 存在构建一个泄漏引用链。

![image-20181225175136615](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20181225175136615.png)
1. 计算引用泄漏对象大小。bitmap字节数组有时被native gc roots所持有所以不应该包含在这里

![image-20181225175852043](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20181225175852043.png)

最后将分析结果对象序列话到文件。DisplayLeakActivity访问时去load这个对象将信息加载到界面。

![企业微信截图_dfbf285a-1355-4615-b015-050befb5fbf3](/Users/meiweibuyondeng/Library/Containers/com.tencent.WeWorkMac/Data/Library/Application Support/WXWork/Temp/ScreenCapture/企业微信截图_dfbf285a-1355-4615-b015-050befb5fbf3.png)

[拆轮子系列](https://ivanljt.github.io/blog/2017/12/15/%E6%8B%86%E8%BD%AE%E5%AD%90%E7%B3%BB%E5%88%97%E2%80%94%E2%80%94LeakCanary%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86/)



#####对于线上使用需求的一些问题及建议

1. 每次activity或者fragment destory 都会去触发gc,频繁的gc会使应用卡顿。如果不gc单单将监控对象放入KeyedWeakReference那么监控对象只有等下一次gc才会被回收而自动gc的时间是不确定的。

2. 每次出现内存泄漏都会去dump进程内存信息得到一个hprof文件 这个文件的大小起码是几十M的大小

   需要注意设备磁盘的使用。leakCannary对这块的处理。文件不能超过7个。10分钟内dump过不进行dump。

   对hprof文件分析是比较耗时和占用cpu的。所以不能在用户使用应用时进行分析。

3. leakCannary主要对activity，fragment对象进行监控，这个比较容易因为我们知道onDestroy后对象就应该被回收所以选择这个时候去分析。而此次需求是要求对泄漏的对象进行监控。如果不知道什么对象什么时候应该回收就比较难分析了。

4. 我建议这次客户端只负责加一个dump内存的入口。设备有问题让技术人员或者用户点击dump上传hprof文件，hprof文件可以在studio中直接打开，后续的定位让开发自己猜测分析(因为设备出现问题的状态是不一致的，不知道对象的回收时间)。


