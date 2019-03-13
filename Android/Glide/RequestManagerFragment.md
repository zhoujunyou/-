### RequestManagerFragment

每当调用Glid.wih()，Context，Activity，FragmentActivity，Fragment，android.app.Fragment，View时会去

生成applicationManager或者一个不带界面的RequestManagerFragment并commit。fragment中持有ActivityFragmentLifecycle lifecycle这个lifecycle会被用于构建RequestManager requestManager 并且构建时

 lifecycle.addListener(this)，lifecycle.addListener(connectivityMonitor)添加监听类;

当onStart,onStop,onDestroy方法触发时去调用ActivityFragmentLifecycle的方法回调listener的方法，DefaultConnectivityMonitor回去注册及反注册网络监听广播。网络恢复到连接后会调用requestTracker.restartRequests()

**下面是RequestManager的生命周期**，每个applicationManager或RequestManagerFragment持有一个RequestManager。每个RequestManager有一个RequestTracker用于追踪Request.以及各个生命周期下对

Request的处理.

```java
@Override
public void onStart() {
  resumeRequests();
  targetTracker.onStart();
}

/**
 * Lifecycle callback that unregisters for connectivity events (if the
 * android.permission.ACCESS_NETWORK_STATE permission is present) and pauses in progress loads.
 */
@Override
public void onStop() {
  pauseRequests();
  targetTracker.onStop();
}

/**
 * Lifecycle callback that cancels all in progress requests and clears and recycles resources for
 * all completed requests.
 */
@Override
public void onDestroy() {
  targetTracker.onDestroy();
  for (Target<?> target : targetTracker.getAll()) {
    clear(target);
  }
  targetTracker.clear();
  requestTracker.clearRequests();
  lifecycle.removeListener(this);
  lifecycle.removeListener(connectivityMonitor);
  mainHandler.removeCallbacks(addSelfToLifecycle);
  glide.unregisterRequestManager(this);
}
```

以上的这些逻辑是Glide同步生命周期处理的机制。

