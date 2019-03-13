### HandlerThread

了解了Handler机制后 HandlerThread其实比较简单

```
/**
 * Handy class for starting a new thread that has a looper. The looper can then be 
 * used to create handler classes. Note that start() must still be called.
 */
```

1. 创建了一个线程
2. 帮忙开启了Looper
3. 提供quit和quitSafe方法

```java
  @Override
    public void run() {
        mTid = Process.myTid();
        Looper.prepare();
        synchronized (this) {
            mLooper = Looper.myLooper();
            notifyAll();
        }
        Process.setThreadPriority(mPriority);
        onLooperPrepared();
        Looper.loop();
        mTid = -1;
    }
```

```java
public Looper getLooper() {
        if (!isAlive()) {
            return null;
        }
        
        // If the thread has been started, wait until the looper has been created.
        synchronized (this) {
            while (isAlive() && mLooper == null) {
                try {
                    wait();
                } catch (InterruptedException e) {
                }
            }
        }
        return mLooper;
    }


```

```java
 /**
     * Call back method that can be explicitly overridden if needed to execute some
     * setup before Looper loops.
     */
//在loop开启前 会在当前线程调用该方法
    protected void onLooperPrepared() {
    }
```



#### 简单使用

```java
HandlerTread handlerThread = new HandlerThread("m");
handlerThread.start();
Handler handler = new Handler(handleThread.getLooper(),callback);
```

