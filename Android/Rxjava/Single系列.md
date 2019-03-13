### Single系列

看了Single系列的代码 这逻辑才符合我的胃口啊 没有并发没有熔合 哈哈自己比较菜吧！

捡起几个经常用的

downstream 下游

Upstream 上游

dispose 丢弃中断

1. **create**

   SingleCreate  

   ```java
    @Override
           public void onSuccess(T value) {
               if (get() != DisposableHelper.DISPOSED) {
                   Disposable d = getAndSet(DisposableHelper.DISPOSED);
                   if (d != DisposableHelper.DISPOSED) {
                       try {
                           if (value == null) {
                               downstream.onError(new NullPointerException("onSuccess called with null. Null values are generally not allowed in 2.x operators and sources."));
                           } else {
                               downstream.onSuccess(value);
                           }
                       } finally {
                           if (d != null) {
                               d.dispose();
                           }
                       }
                   }
               }
           }
   ```
   判断也较简单 流被dispose了就不处理 否则就直接调用下游的方法 调用完dispose

2. **just**

   SingleJust    更简单了

   ```java
     @Override
       protected void subscribeActual(SingleObserver<? super T> observer) {
           observer.onSubscribe(Disposables.disposed());
           observer.onSuccess(value);
       }
   ```


3. **delay**

​      SingleDelay

   ```java
     @Override
           public void onSuccess(final T value) {
               sd.replace(scheduler.scheduleDirect(new OnSuccess(value), time, unit));
           }
   ```

   就是将上游的数据放到线程中延迟执行。 replace也用于downstream中有没dispose 若dispose了当前 该处是scheuler也会被dispose

4. **map**
  SingleMap

   ```java
    @Override
           public void onSuccess(T value) {
               R v;
               try {
                   v = ObjectHelper.requireNonNull(mapper.apply(value), "The mapper function returned a null value.");
               } catch (Throwable e) {
                   Exceptions.throwIfFatal(e);
                   onError(e);
                   return;
               }
   
               t.onSuccess(v);
           }
   ```

   map也很简单。

5. **flatmap**
   SingleFlatMap

   ```java
    @Override
           public void onSuccess(T value) {
               SingleSource<? extends R> o;
   
               try {
                   o = ObjectHelper.requireNonNull(mapper.apply(value), "The single returned by the mapper is null");
               } catch (Throwable e) {
                   Exceptions.throwIfFatal(e);
                   downstream.onError(e);
                   return;
               }
   
               if (!isDisposed()) {
                   o.subscribe(new FlatMapSingleObserver<R>(this, downstream));
               }
           }
   ```
   可以看到flatmap 会在upstream调用onSucces时重新创建了个Single 再去subscribe downstream

6. **compose**

    SingleFromUnsafeSource

   ```java
   public final <R> Single<R> compose(SingleTransformer<? super T, ? extends R> transformer) {
           return wrap(((SingleTransformer<T, R>) ObjectHelper.requireNonNull(transformer, "transformer is null")).apply(this));
       }
   ```

   ```java
       @Override
       protected void subscribeActual(SingleObserver<? super T> observer) {
           source.subscribe(observer);
       }
   ```

   个人理解compose操作功能 就是对操作符组合的封装(类似平常我们将一些常用功能封装到一个方法中)

   compose经常被拿来和flatmap做比较 这里自己根据自己的理解也记录一些不同点

| 不同点           | compose                         | flatmap                                                      |
| ---------------- | ------------------------------- | ------------------------------------------------------------ |
| 内部接口执行时机 | 方法调用时                      | upsteam调用onSuccess时                                       |
| 对流的影响       | 还是原来的逻辑 一直向上subscibe | upsteam调用onSuccess时 会创建一个新的Single去subscribe下游的observer |

   上面第二点会有什么印象呢 举个栗子 在compose中调用subscribeOn等还会影响整个流

   而在flatmap中调用只会影响flatmap中的流。

7. **lift**

    SingleOperator

   ```java
   @NonNull
       SingleObserver<? super Upstream> apply(@NonNull SingleObserver<? super Downstream> observer) throws Exception;
   ```
    SingleLift

   ```java
    protected void subscribeActual(SingleObserver<? super R> observer) {
           SingleObserver<? super T> sr;
   
           try {
               sr = ObjectHelper.requireNonNull(onLift.apply(observer), "The onLift returned a null SingleObserver");
           } catch (Throwable ex) {
               Exceptions.throwIfFatal(ex);
               EmptyDisposable.error(ex, observer);
               return;
           }
   
           source.subscribe(sr);
       }
   ```

	SignleLift原理就是在subscribe下游observer时 替换成SingleOperator转换后的observer 其实就是将处理observer的方法交给用户自己实现

8. timer

    SingleTimer

   ```java
   @Override
       protected void subscribeActual(final SingleObserver<? super Long> observer) {
           TimerDisposable parent = new TimerDisposable(observer);
           observer.onSubscribe(parent);
           parent.setFuture(scheduler.scheduleDirect(parent, delay, unit));
       }
   
     @Override
           public void run() {
               downstream.onSuccess(0L);
           }
   ```

   timer也简单 延迟相应时间发个0L

9. **defer**

    SingleDefer

   ```java
   @Override
       protected void subscribeActual(SingleObserver<? super T> observer) {
           SingleSource<? extends T> next;
   
           try {
               next = ObjectHelper.requireNonNull(singleSupplier.call(), "The singleSupplier returned a null SingleSource");
           } catch (Throwable e) {
               Exceptions.throwIfFatal(e);
               EmptyDisposable.error(e, observer);
               return;
           }
   
           next.subscribe(observer);
       }
   ```

   简单 延迟Single的创建 调用subscribe 才去创建真正的Single

10. 