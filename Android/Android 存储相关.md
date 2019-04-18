###Android 存储相关

需求：最近项目中有因为有db过大导致项目无法正常运行的问题 。所以利用网管监控项目采集客户端各个db的大小。

#### 1. 内部存储

**/data/data/package_name/ (会随着应用的卸载一起删除掉)**  Context.getCacheDir().getParentFile()对应这个目录 以下是子目录

android 6.0对应**/data/user/0/package_name/**

这两条绝对路径指向同一个文件。这是因为 Android 6.0 支持多用户，文件的实际路径是 /data/user/0/XXX（0代表的用户ID），而 /data/data/XXX 是为了方便查看数据而创建的引用目录，实际指向 /data/user/0/XXX，下文有时会出现 /data/data/XXX 和 /data/user/0/XXX表述的是一样的意思，如果是 Android 5.0 及以下版本，返回的是 /data/data/XXX, 这下完全明白了。


| 路径          | 类型                 | 方法                              |
| ------------- | -------------------- | --------------------------------- |
| /shared_prefs | 存储SharedPreference |                                   |
| /databases    | 存储数据库DB         | **getDatabasePath()**             |
| /app_webview  | 存储webview相关      |                                   |
| /lib          | 存储.so静态库文件    |                                   |
| /cache        | 存放临时             | getCacheDir()                     |
| /custom       | 创建自己的目录       | **getDir(String name, int mode)** |
| /files        |                      | getFilesDir()                     |





#### 2. 外部存储

 	##### （1）SD卡

   获取路径方式是`Environment.getExternalStorageDirectory()` `/storage/sdcard0`

| 路径                           | 方法                                                         |      |
| ------------------------------ | ------------------------------------------------------------ | ---- |
| /storage/sdcard0/Alarms        | Environment.getExternalStoragePublicDirectory(DIRECTORY_ALARMS) |      |
| /storage/sdcard0/DCIM          | Environment.getExternalStoragePublicDirectory(DIRECTORY_DCIM) |      |
| /storage/sdcard0/Download      | Environment.getExternalStoragePublicDirectory(DIRECTORY_DOWNLOADS) |      |
| /storage/sdcard0/Movies        | Environment.getExternalStoragePublicDirectory(DIRECTORY_MOVIES) |      |
| /storage/sdcard0/Music         | Environment.getExternalStoragePublicDirectory(DIRECTORY_MUSIC) |      |
| /storage/sdcard0/Notifications | Environment.getExternalStoragePublicDirectory(DIRECTORY_NOTIFICATIONS) |      |
| /storage/sdcard0/Pictures      | Environment.getExternalStoragePublicDirectory(DIRECTORY_PICTURES) |      |
| /storage/sdcard0/Podcasts      | Environment.getExternalStoragePublicDirectory(DIRECTORY_PODCASTS) |      |
| /storage/sdcard0/Ringtones     | Environment.getExternalStoragePublicDirectory(DIRECTORY_RINGTONES) |      |

**上面的九个方法对应的就是SD卡的九大公有目录，Google官方建议我们数据应该存储在私有目录下，不建议存储在公有目录下或其他地方**



   私有目录`/storage/emulated/0/Android/data/package_name/files`

| 路径                                                | 方法                  |
| --------------------------------------------------- | --------------------- |
| /storage/emulated/0/Android/data/package_name/files | getExternalFilesDir() |
| /storage/emulated/0/Android/data/package_name/cache | getExternalCacheDir   |

  ##### （2）扩展卡内存 

StroageVolume 中的方法getVolumeList()被隐藏了需要反射调用

```java
private static String getExtendedMemoryPath(Context mContext) {  
      StorageManager mStorageManager = (StorageManager) mContext.getSystemService(Context.STORAGE_SERVICE);
        Class storageVolumeClazz = null;
        try {
            storageVolumeClazz = Class.forName("android.os.storage.StorageVolume");
            Method getVolumeList = mStorageManager.getClass().getMethod("getVolumeList");
            Method getPath = storageVolumeClazz.getMethod("getPath");
            Method isRemovable = storageVolumeClazz.getMethod("isRemovable");
            Object result = getVolumeList.invoke(mStorageManager);
            final int length = Array.getLength(result);
            for (int i = 0; i < length; i++) {
                Object storageVolumeElement = Array.get(result, i);
                String path = (String) getPath.invoke(storageVolumeElement);
                boolean removable = (Boolean) isRemovable.invoke(storageVolumeElement);
                if (removable) {
                    return path;
                }
            }
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
        return null;
}
```

***

以上是对android应用存储的一个学习

 ##### 获取db文件大小

```java
 public void getDataBaseInfo(Context context) {
        File parentFile = context.getCacheDir().getParentFile();
        Log.i(TAG, "parentFile=" + parentFile.getAbsolutePath());
        File dataBaseFile = new File(parentFile, "databases");
        List<File> files = FileUtils.listFilesInDirWithFilter(dataBaseFile, new FileFilter() {
            @Override
            public boolean accept(File pathname) {
                return pathname.getName().endsWith(".db");
            }
        },false);
        if (files != null && files.size() > 0) {
            for (File file : files) {
                Log.i(TAG, "file=" + file.getName());
                Log.i(TAG, "size=" + FileUtils.getFileSize(file));
            }
        }
    }
```

[FileUits](https://github.com/Blankj/AndroidUtilCode/blob/master/utilcode/src/main/java/com/blankj/utilcode/util/FileUtils.java)









##### 获取app存储信息

1. 查看了android 6.0设置应用的源码 最后是通过android.content.pm.PackageManager.getPackageSizeInfo方法拿到了PackageStats这个对象

   ```java
    /** Name of the package to which this stats applies. */
       public String packageName;
   
       /** @hide */
       public int userHandle;
   
       /** Size of the code (e.g., APK) */
       public long codeSize;
   
       /**
        * Size of the internal data size for the application. (e.g.,
        * /data/data/<app>)
        */
       public long dataSize;
   
       /** Size of cache used by the application. (e.g., /data/data/<app>/cache) */
       public long cacheSize;
   
       /**
        * Size of the secure container on external storage holding the
        * application's code.
        */
       public long externalCodeSize;
   
       /**
        * Size of the external data used by the application (e.g.,
        * <sdcard>/Android/data/<app>)
        */
       public long externalDataSize;
   
       /**
        * Size of the external cache used by the application (i.e., on the SD
        * card). If this is a subdirectory of the data directory, this size will be
        * subtracted out of the external data size.
        */
       public long externalCacheSize;
   
       /** Size of the external media size used by the application. */
       public long externalMediaSize;
   
       /** Size of the package's OBBs placed on external media. */
       public long externalObbSize;
   ```

2. 因为我们应用无法调用到上面的方法 所以用反射实现

   ```java
   public void getAppStorageSize(final SizeCallBack callBack) throws Exception {
           Class<?> loadClass = this.getClass().getClassLoader().loadClass("android.content.pm.PackageManager");
           Method method = loadClass.getDeclaredMethod("getPackageSizeInfo", String.class, IPackageStatsObserver.class);
           //receiver : 类的实例,隐藏参数,方法不是静态的必须指定
           method.invoke(Utils.getApp().getPackageManager(), Utils.getApp().getPackageName(), new IPackageStatsObserver.Stub() {
               @Override
               public void onGetStatsCompleted(PackageStats stats, boolean succeeded) {
                   long externalCodeSize = stats.externalCodeSize
                           + stats.externalObbSize;
                   long externalDataSize = stats.externalDataSize
                           + stats.externalMediaSize;
                   long newSize = externalCodeSize + externalDataSize
                           + stats.codeSize + stats.dataSize;
   //                        long cachesize = stats.cacheSize;//缓存大小
   //                        long codesize = stats.codeSize;//应用程序的大小
   //                        long datasize = stats.dataSize;//数据大小
                   callBack.onSizeCallBack(newSize);
               }
           });
       }
   
       public interface SizeCallBack {
           void onSizeCallBack(long size);
       }
   ```

3. 记得在aidl中添加以下两个aidl  IPackageStatsObserver.aidl ,PackageStats.aidl

   ```java
   package android.content.pm;
   
   import android.content.pm.PackageStats;
   /**
    * API for package data change related callbacks from the Package Manager.
    * Some usage scenarios include deletion of cache directory, generate
    * statistics related to code, data, cache usage(TODO)
    * {@hide}
    */
   oneway interface IPackageStatsObserver {
       
       void onGetStatsCompleted(in PackageStats pStats, boolean succeeded);
   }
   
   ```

   ```java
   package android.content.pm;
   
   parcelable PackageStats;
   ```



