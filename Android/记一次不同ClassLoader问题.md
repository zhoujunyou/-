### 记一次不同ClassLoader问题

业务中心日志打点明明时单例却有多个实例
MweeLogInner 的init方法

```java
 private static LoggerHandler logHandler;
 private static LogRepository logDBUtil;
synchronized (MwLogInner.class) {
            if (inited && !TextUtils.isEmpty(shopId) && !TextUtils.isEmpty(hostId)) {
                return;
            }
            inited = true;
        }
```

这样写应该只会被init一次 debug的时候却有多个logDBUtil和logHandler实例 怀疑和插件有关系。

![image-20190130121942651](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20190130121942651.png)

![image-20190130122012793](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20190130122012793.png)
分别打印出ClassLoader 可以看到确实不是同一个classloader



[深入理解Android中的ClassLoader](https://juejin.im/post/5a28e7e86fb9a045117105c3)

ClassLoader双亲加载机制：先看该类有没加载, 没加载先让parent去加载 parent加载不到再自己去加载.

DexClassLoader和pathClassLoader都是派生自BaseClassLoader ：

 **DexClassLoader有个optimizedDirectory参数可以加载任何路径的apk/dex/jar ， PathClassLoader没有optimizedDirectory，所以它只能加载内部的dex，这些大都是存在系统中已经安装过的apk里面的，比如/data/app中的apk。这个也是PathClassLoader作为默认的类加载器的原因，因为一般程序都是安装了，在打开，这时候PathClassLoader就去加载指定的apk(解压成dex，然后在优化成odex)就可以了**



所以很有可能是com.mwee.plugin.PluginClassLoader加载的是插件apk中的dex,而系统的PathClassLoader 加载的是base apk中的dex  这就是产生不同实例的原因(根本不是同一个类)**一个类，由不同的类加载器实例加载的话，会在方法区产生两个不同的类，彼此不可见，并且在堆中生成不同Class实例。因此单例也是相互隔离的**

---
**分析**

DinnerApplication

```java
 @Override
    public void onCreate() {
        //需要阻塞
        AppLog.initConfig(this);
        //启动业务中心
        if (ProcessUtil.isMainProcess(this)) {
            if (!(ClientBindProcessor.isActived() && !ClientBindProcessor.isCurrentHostMain())) {
                WakeUpBizCenter.callServerProcess(this);
                ServerConnector.getInstance().initConnection(this);
            }
        }
    }
```

* 这里绑定到ServerService 服务开启bizcenter进程重新走到oncreate方法 这里 AppLog.initConfig(this)中的类 由系统的类加载器加载 

* ServerSevice onCreate中`loader = MydPluginLoader.getInstance(getApplicationContext()); `检查插件更新

*  loadPlugin()->loadApl()->active()->createPluginProxy()



   PluginProcessor

  ```java
  protected void loadApk(Context context, String apkPath) {
          // 4.1以后不能够将optimizedDirectory设置到sd卡目录， 否则抛出异常.
          File optimizedDirectoryFile = context.getDir("dex", 0);
          classLoader = new PluginClassLoader(apkPath, optimizedDirectoryFile.getAbsolutePath(),
                  null, context.getClassLoader());
  
          prepareResouce(context, apkPath);
  
      }
  ```

  `serverProxy = pluginManger.reflectNewObject(CLZ_PROXY);`

  ```java
   protected Object reflectNewObject(String clz) throws ClassNotFoundException, NoSuchMethodException, InstantiationException, IllegalAccessException, InvocationTargetException {
          Class<?> mLoadClass = classLoader.loadClass(clz);
          Constructor constructor = mLoadClass.getConstructor();
          return constructor.newInstance();
      }
  ```

  所以ServerProxy是由自定义的ClassLoader加载的.之后通过反射调用ServerProxy.work到BizCenterApplication.initEnv 其中的ServerLog.initConfig(context)也是由PluginClassLoader从dex中加载的

**解决**

   DinnerApplication中先做判断主进程和打印init，业务中心进程不init  

