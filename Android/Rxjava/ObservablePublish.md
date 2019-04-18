

### ObservablePublish&ObservableRefCount

对应操作符：share , publish , refCount

share=publish().refCount();

```java
@Override
    public void connect(Consumer<? super Disposable> connection) {
        boolean doConnect;
        PublishObserver<T> ps;
        // we loop because concurrent connect/disconnect and termination may change the state
        for (;;) {
            // retrieve the current subscriber-to-source instance
            ps = current.get();
            // if there is none yet or the current has been disposed
            if (ps == null || ps.isDisposed()) {
                // create a new subscriber-to-source
                PublishObserver<T> u = new PublishObserver<T>(current);
                // try setting it as the current subscriber-to-source
                if (!current.compareAndSet(ps, u)) {
                    // did not work, perhaps a new subscriber arrived
                    // and created a new subscriber-to-source as well, retry
                    continue;
                }
                ps = u;
            }
            // if connect() was called concurrently, only one of them should actually
            // connect to the source
            doConnect = !ps.shouldConnect.get() && ps.shouldConnect.compareAndSet(false, true);
            break; // NOPMD
        }
        try {
            connection.accept(ps);
        } catch (Throwable ex) {
            Exceptions.throwIfFatal(ex);
            throw ExceptionHelper.wrapOrThrow(ex);
        }
        if (doConnect) {
            source.subscribe(ps);
        }
    }
```

```java
  @Override
        public void dispose() {
            Object o = getAndSet(this);
            if (o != null && o != this) {
                ((PublishObserver<T>)o).remove(this);
            }
        }
```



每次subscribe observer时 将 observer添加到PublishObserver中 并没有将上游source与下游observer关联
下游调用dispose 仅移除相应observer。

直到调用了connect方法  并判断已连接才会调用到source.subscribe(observer)



```java
   @SuppressWarnings("unchecked")
        @Override
        public void dispose() {
            InnerDisposable[] ps = observers.getAndSet(TERMINATED);
            if (ps != TERMINATED) {
                current.compareAndSet(PublishObserver.this, null);

                DisposableHelper.dispose(upstream);
            }
        }
```

 connection.accept(ps)用于将dispose对象返回到外层调用处  ps调用dispose则直接dispose上游数据.



---
####ObservableRefCount
```
@Override
protected void subscribeActual(Observer<? super T> observer) {

    RefConnection conn;

    boolean connect = false;
    synchronized (this) {
        conn = connection;
        if (conn == null) {
            conn = new RefConnection(this);
            connection = conn;
        }

        long c = conn.subscriberCount;
        if (c == 0L && conn.timer != null) {
            conn.timer.dispose();
        }
        conn.subscriberCount = c + 1;
        if (!conn.connected && c + 1 == n) {
            connect = true;
            conn.connected = true;
        }
    }

    source.subscribe(new RefCountObserver<T>(observer, this, conn));

    if (connect) {
        source.connect(conn);
    }
}
```

  if (!conn.connected && c + 1 == n)   observer对象个数达到n时会调用 source.connect(conn);