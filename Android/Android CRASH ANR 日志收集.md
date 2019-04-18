#### Android CRASH ANR 日志收集

需求：如图

![image-20180919183318650](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20180919183318650.png)

#### 收集实现方式

1. 通过DropBoxManager收集（直接读取/data/system/dropbox文件)
   ![image-20180919173220331](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20180919173220331.png)

2. 通过现成的bugly收集 在bugly上报crash 或anr之前先将错误信息写入本地磁盘

   ```java
    /**
                * Crash处理.
                *
                * @param crashType 错误类型：CRASHTYPE_JAVA，CRASHTYPE_NATIVE，CRASHTYPE_U3D ,CRASHTYPE_ANR
                * @param errorType 错误的类型名
                * @param errorMessage 错误的消息
                * @param errorStack 错误的堆栈
                * @return 返回额外的自定义信息上报
                */
               @Override
               public Map<String, String> onCrashHandleStart(int crashType, String errorType,
                                                             String errorMessage, String errorStack) {
                   LinkedHashMap<String, String> map = new LinkedHashMap<String, String>();
                   map.put("设备序列号","test");
                   Log.i("bugly_test",crashType+errorType+errorMessage);
                   return map;
               }
   ```


不过目前上述方案都存在一些问题

* 第一种方案 只有系统app才有权限访问该文件夹。查看了sunmi sendcrash的实现(这个是系统应用 所以能读取到/data/system/dropbox)。

  1. 监听到日志被添加的广播->读取/data/system/dropbox内容->写入外部存储/sdcard/crash.log文件

     所以说summi上装了sendcrash应用 美易点就在crash.log中能读取到需要收集的信息

* 第二种方案：系统崩溃都能获取到 但是一个很尴尬的地方Android6.0后的设备无法上传ANR信息

  还是由于系统权限的限制导致bugly 无法查看trace.txt文件引起。不过这部分信息非常关键 分析线上的一些特别异常问题就需要这部份信息.

#### 上报方式

1. 将日志写入文件上报中控点 击多次一条条上报   不过要做好一些限制和考虑一些会存在的问题 
   * 文件的大小是否要做下限制(太大了不行)

   * 同一条记录是否允许多次上报

   * 多条上报时一条失败是否都终止(避免对服务器造成压力)

   * 这样一条条上报其实对查询日志不友好 后台日志会多很多zip日志包


####界面

1. 对日志的条数和时间做下限制。



#### 建议

1. 关于收集 建议是设备厂商给我们开放一下读取权限 目前只能在商米设备上实现收集需要的全部信息

2. 关于上报  根据上文上报方式里的问题

      建议：其实上面的上报日志一些问题不管在美易点还是全能POS都有自己的解决方案和实现方式 
      1.上报日志的时候讲这些日志添加到应用日志中

   2.美易点本身有上报到KB的逻辑 这个查起来也很方便 

   3.如果还是这个方案 建议和中控其他日志分开不然后面查询过滤比较麻烦 

3. 关于界面 限制日志条数和时间来提升交互效果及体验（太多影响内存）