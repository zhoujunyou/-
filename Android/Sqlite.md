### Android SqliteDataBase

* Sqlite是线程安全吗？

  看下SqliteDatabase insert过程

   insert->insertWithOnConflict-SQLiteStatement.executeInsert()->SQLiteSession.executeForLastInsertedRowId->SQLiteConnection.executeForLastInsertedRowId->nativeExecuteForLastInsertedRowId

  ```
  // Thread-local for database sessions that belong to this database.
  // Each thread has its own database session.
  // INVARIANT: Immutable.
  private final ThreadLocal<SQLiteSession> mThreadSession = new ThreadLocal<SQLiteSession>() {
      @Override
      protected SQLiteSession initialValue() {
          return createSession();
      }
  };
  ```

  SQLiteSession 为线程独有共用一个SQLiteConnectionPool  SQLiteConnectionPool的acquireConnection方法回去申请SqliteConnections 如果被占用会维护一个线程mConnectionWaiterQueue等待。等占用线程执行完调用releaseConnection  唤醒等待队列头部线程 ，所以说SqliteDataBase是单例的情况下是线程安全的。注意点。其他excute方法中SQLiteSession会在finally中调用releaseConnection 而beginTransaction需要手动调用endTransaction 。

  **关于attempt to re-open an already-closed object多线程异常的原因**： 没有close之前SQLiteOpenHelper.getWriteBase方法时返回的是同一个database对象 mReferenceCount初始值为1  SqliteDataBase中的close方法用于减少mReferenceCount 当mReferenceCount减少到0的的时候去释放连接mConnectionPoolLocked=null 。所以当两个线程先调用getWriteBase 后一个线程调用close 那另外一个线程就容易抛出这个异常。

  **所以在使用单例的SqliteHelper和不调用close方法多线程调用是没有问题的，用完调用close可以用计数的方法做一下保护**  

* **android.database.CursorWindowAllocationException Cursor window allocation of 2048 kb failed**.

  
  具体抛出异常的代码 是在CursorWindow的构造方法中 mWindowPtr用于native中寻址。

  ```java
   if (sCursorWindowSize < 0) {
              /** The cursor window size. resource xml file specifies the value in kB.
               * convert it to bytes here by multiplying with 1024.
               */
              sCursorWindowSize = Resources.getSystem().getInteger(
                  com.android.internal.R.integer.config_cursorWindowSize) * 1024;
          }
          mWindowPtr = nativeCreate(mName, sCursorWindowSize);
          if (mWindowPtr == 0) {
              throw new CursorWindowAllocationException("Cursor window allocation of " +
                      (sCursorWindowSize / 1024) + " kb failed. " + printStats());
          }
       mCloseGuard.open("close");
          recordNewWindow(Binder.getCallingPid(), mWindowPtr);
  ```

  抛出这个异常的原因是 一个窗口的大小需要2M内存 这里拿不到足够的资源 原因很可能是用完curson后没有close 方法 。还有可能就是fd被用完了,**使用 shell ls -l /proc/$pidnum/fd命令打印出fd 可以看到/dev/ashmem在增加**, CursorWindow.cpp的create方法中会去调用 ashmemFd=ashmem_create_region(ashmemName.string(), size);返回的ashmemFd如果小于0就报异常了 (比如项目中打印进程中这种问题比较多,这个问题可以通过fd监控来进行定位) 

  1. 是如何申请一个nativeWindow的

     调用SqliteDataBase rawQuery方法 新建一个SQLiteDirectCursorDriver用于query  在这里我们的CursorFactory一般都为null  所以新建一个SQLiteCursor  并返回.

     当调用SQLiteCursor的moveToNext 会走到父类AbstractCursor的moveToPosition再走到SQLiteCursor的onMove中 这里第一次调用我们的mWindow属性为null 进入fillWindow  走到clearOrCreateWindow 新建一个CursorWindow   **构造方法就是上面的代码用于在native中申请一块地址mWindowPtr就相当于指针，之后查询字段及回收都需要用到这个值**，将mWindow作为参数传入SQLiteQuery的fillWindow 再到SQLiteSession到 SQLiteConnection 最后调用nativeExecuteForCursorWindow 将window的mWindowPtr 传到native层 

  2. 释放window

     cursor调用close ->releaseReference->CursorWindow.onAllReferencesReleased->dispose->nativeDispose(mWindowPtr)。
     CursorWindow的finalize方法也会去dispose 

  ---

  项目中有类似这样的调用 

  ```java
  Cursor cursor = null;
  try {
      cursor = db.rawQuery("SELECT * FROM log ", null);
      cursor.moveToNext();
      cursor = db.rawQuery("SELECT * FROM log ", null);
      cursor.moveToNext();
      cursor = db.rawQuery("SELECT * FROM log ", null);
      cursor.moveToNext();
  } finally {
      cursor.close();
      Log.i(TAG, "cursor leak test size :" + sWindowToPidMap.size() + " stats" + printStats());
  }
  ```

  这样并不能关闭所有cursor,因为每次db.rawQuery都为新建一个CursorWindow 带来新的mWindowPtr 

  所以这样只能回收最后一个cursor在native中的window 而前面的curson对应native中的window相当于没回收. 当gc触发时才有机会去回收native的内存 





---
[文档](https://time.geekbang.org/column/article/77546)

**以下内容从极客时间拷贝**

SQLite 默认是支持多进程并发操作的，它通过文件锁来控制多进程的并发。SQLite 锁的粒度并没有非常细，它针对的是整个 DB 文件，内部有 5 个状态。
简单来说，多进程可以同时获取 SHARED 锁来读取数据，但是只有一个进程可以获取 EXCLUSIVE 锁来写数据库。
在 EXCLUSIVE 模式下，数据库连接在断开前都不会释放 SQLite 文件的锁，从而避免不必要的冲突，提高数据库访问的速度。

相比多进程，多线程的数据库访问可能会更加常见。SQLite 支持多线程并发模式，需要开启下面的配置，当然系统 SQLite 会默认开启多线程
跟多进程的锁机制一样，为了实现简单，SQLite 锁的粒度都是数据库文件级别，并没有实现表级甚至行级的锁 。还有需要说明的是，同一个句柄同一时间只有一个线程在操作，，这个时候我们需要打开连接池 Connection Pool。
跟多进程类似，多线程可以同时读取数据库数据，但是写数据库依然是互斥的。SQLite 提供了 Busy Retry 的方案，即发生阻塞时会触发 Busy Handler，此时可以让线程休眠一段时间后，重新尝试操作，你可以参考[微信iOS SQLite源码优化实践](https://mp.weixin.qq.com/s/8FjDqPtXWWqOInsiV79Chg)
为了进一步提高并发性能，我们还可以打开WAL（Write-Ahead Logging）模式。WAL 模式会将修改的数据单独写到一个 WAL 文件中，同时也会引入了 WAL 日志文件锁。通过 WAL 模式读和写可以完全地并发执行，不会互相阻塞。但是需要注意的是，写之间是仍然不能并发,。如果出现多个写并发的情况，依然有可能会出现 SQLiteDatabaseLockedException。这个时候我们可以让应用中捕获这个异常，然后等待一段时间再重试。

```java
} catch (SQLiteDatabaseLockedException e) {  
    if (sqliteLockedExceptionTimes < (tryTimes - 1)) { 
        try {          
            Thread.sleep(100);    
        } catch (InterruptedException e1) { 
            
        }    
    }    sqliteLockedExceptionTimes++；
}
```



总的来说通过连接池与 WAL 模式，我们可以很大程度上增加 SQLite 的读写并发，大大减少由于并发导致的等待耗时，建议大家在应用中可以尝试开启。



