###神策Android全埋点方案

原理简单分析:    Activity生命周期通过监听Application.ActivityLifecycleCallbacks，fragment的生命周期 及一些点击事件则编译时通过ASM对相应方法进行hook

#### 神策Android SDK分析

[sdk git仓库](https://github.com/sensorsdata/sa-sdk-android)  

[官网SDK介绍](https://www.sensorsdata.cn/manual/android_sdk.html)

##### Gradle 插件分析

仓库上好像没有插件代码,通过http://jcenter.bintray.com/com/sensorsdata/analytics/android/android-gradle-plugin2/2.0.0/ 下载相应jar包解压

![image-20180916112522297](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20180916112522297.png)目录

![sa-gradle](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/sa-gradle.png)gradle plugin uml

1. SensorsAnalyticPlugin 插件入口
2. SensorsAnalyticsExtension 配置文件（debug 是否输出日日志,disableJar是否修改jar包,exclude不修改的包）
3. SensorsAnalyticsTransform 遍历jar 遍历目录满足条件调用SensorsAnalyticsClassVisitor
4. SensorsAnalyticsClassVisitor 扫描到SensorsAnalyticsHookConfig 中配置的方法时字节码修改调用sdk中com/sensorsdata/analytics/android/sdk/SensorsDataAutoTrackHelper的方法



#### SDK 分析

1. org.aspectj:aspectjrt:1.8.10  实际上并没用到 其实用的是上面的ASM 所以可以去除这个依赖以及  com.sensorsdata.analytics.android.sdk.aop这个包

2.  AnalyticsMessages 类用于上报。逻辑简单看了下 开了个Work线程。直接上报 或者间隔一端时间去上报。
3.  TrackTaskManager 任务列表(每次track都是个任务)对应TrackTaskManagerThread 
4. TrackTaskManagerThread 这是个Runable 里面开了个单线程线程池每个3秒去 任务列表拉任务并执行

**关键的类和方法**

1. SensorsDataAutoTrackHelper 用于v4/Fragment生命周期和各种视图事件的track 和插件 SensorsAnalyticsHookConfig中相对应
2. SensorsDataActivityLifecycleCallbacks  Activity生命周期track
3.  SensorsDataAPI如下方法（主要附加了一些当前环境数据的track  每次track都需要走到这里这个可能比较耗时)
```java
   private void trackEvent(final EventType eventType, final String eventName, final JSONObject properties, final String
           originalDistinctId)
```
### 总结

1. 涉及的业务的还是需要额外写入代码 比如点击按钮也只能获取到当前页面和按钮上的文字

   对于B端比较关注一些业务数据的不合适 还需要寻找新的解决方法

2. 每次页面操作和点击事件都会去额外执行的方法需要 测下耗时 还有打点前后对app性能的影响

3. sdk采集数据可以参考上面的关键方法和类。 打包插件可以参考fork一份 方便自己配置需要Hook哪些方法

#### 额外知识

1. handle是否提交了相应任务可以用这个方法判断 

```
/**
     * Check if there are any pending posts of messages with code 'what' in
     * the message queue.
     */
    public final boolean hasMessages(int what) {
        return mQueue.hasMessages(this, what, null);
    }
```

