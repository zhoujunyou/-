1. adb shell top命令

   1. 命令功能：top命令提供了实时的对系统处理器的状态监视.它将显示系统中CPU最“敏感”的任务列表.该命令可以按CPU使用.内存使用和执行时间对任务进行排序.

2. ps

   1.ps命令用来列出系统中当前运行的那些进程。ps命令列出的是当前那些进程的快照，就是执行ps命令的那个时刻的那些进程，如果想要动态的显示进程信息，就可以使用top命令[学习](http://www.cnblogs.com/peida/archive/2012/12/19/2824418.html)



#### 获取已安装apk例子

```shell
adb shell pm list packages|grep send
adb shell pm path com.sunmi.sendcrash
adb pull /data/app/com.sunmi.sendcrash-1/base.apk /Users/meiweibuyondeng/Downloads
```



