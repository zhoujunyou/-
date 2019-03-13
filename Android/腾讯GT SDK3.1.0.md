

### 腾讯GT SDK3.1.0

app集成gt_sdk用与采集性能相关数据 通过aidl进程间通信将数据上送到gt_app

#### 1.数据采集

1. 应用包名,pid 通过绑定服务intent传递
2. NormalMonitor基础信息收集 cpu 内存 流量等
3. ScreenMonitor 广播监听屏幕休眠及活动状态收集
4. LogcatMonitor 日志信息通过logcat收集
5. ChoreographerMonitor 通过postFrameCallback 收集FPS信息
6. HookMonitor 通过[YAHFA--ART](http://rk700.github.io/2017/03/30/YAHFA-introduction/) native hook方式获取activity fragment生命周期 view创建时间 绘制时间 db插入线程等信息。 



### 额外

1.判断一个应用是否安装 gt用的这种方式

```java
/**
	 * 通过被测应用的Context查找GT主应用是否存在，
	 *  如果存在，则返回一个GT应用的Context对象，注意该Context并非GT应用中的对象，
	 * 只是GT应用Context的一个镜像
	 * 
	 * @param hostContext 被测应用的Context
	 * 
	 * @return GT应用是否已安装
	 */
	private boolean isGTInstalled(Context hostContext) {
	    return (getGTContext(hostContext) != null);
	}
	
	private static Context getGTContext(Context context) {
	    Context result = null;
		try {
            result = context.createPackageContext(GTInternal.GT_PACKAGE_NAME,
                    Context.CONTEXT_INCLUDE_CODE | Context.CONTEXT_IGNORE_SECURITY);
		} catch (NameNotFoundException e) {
			Log.d(GTInternal.GT_PACKAGE_NAME, "GT is uninstall.");
		}
		return result;
	}
```

2.判断进程是否包含Application主线程

```java
/**
     * 判断进程是否包含Application主线程
     * @param context
     * @param pid
     * @return
     */
    public static boolean isUIProcess(Context context, int pid) {
        String processName = null;
        ActivityManager mActivityManager = (ActivityManager) context.getSystemService(Context.ACTIVITY_SERVICE);
        for (RunningAppProcessInfo appProcess : mActivityManager.getRunningAppProcesses()) {
            if (appProcess.pid == pid) {
                processName = appProcess.processName;
                break;
            }
        }

        String packageName = context.getPackageName();
        return processName != null && processName.equals(packageName);
    }
```



