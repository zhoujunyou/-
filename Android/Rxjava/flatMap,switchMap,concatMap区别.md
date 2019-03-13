 ### flatMap,switchMap,concatMap区别

1. flatmap 

   ```java
    public void onNext(T t) {
               // safeguard against misbehaving sources
               if (done) {
                   return;
               }
               ObservableSource<? extends U> p;
               try {
                   p = ObjectHelper.requireNonNull(mapper.apply(t), "The mapper returned a null ObservableSource");
               } catch (Throwable e) {
                   Exceptions.throwIfFatal(e);
                   upstream.dispose();
                   onError(e);
                   return;
               }
   
               if (maxConcurrency != Integer.MAX_VALUE) {
                   synchronized (this) {
                       if (wip == maxConcurrency) {
                           sources.offer(p);
                           return;
                       }
                       wip++;
                   }
               }
   
               subscribeInner(p);
           }
   ```


flatmap 会在每次上游调用onNext时 调用 mapper.apply(t)生成一个新的ObservableSource 并去subscribe 下游observer。所以说如果在flatmap中切换了线程 下面observer观察到就不一定是有序的 。 



2. switchmap 

   ```java
    @Override
           public void onNext(T t) {
               long c = unique + 1;
               unique = c;
   
               SwitchMapInnerObserver<T, R> inner = active.get();
               if (inner != null) {
                   inner.cancel();
               }
   
               ObservableSource<? extends R> p;
               try {
                   p = ObjectHelper.requireNonNull(mapper.apply(t), "The ObservableSource returned is null");
               } catch (Throwable e) {
                   Exceptions.throwIfFatal(e);
                   upstream.dispose();
                   onError(e);
                   return;
               }
   
               SwitchMapInnerObserver<T, R> nextInner = new SwitchMapInnerObserver<T, R>(this, c, bufferSize);
   
               for (;;) {
                   inner = active.get();
                   if (inner == CANCELLED) {
                       break;
                   }
                   if (active.compareAndSet(inner, nextInner)) {
                       p.subscribe(nextInner);
                       break;
                   }
               }
           }
   ```


switchmap 也是在上游调用onNext时 调用 mapper.apply(t)生成一个新的ObservableSource 并去subscribe 下游observer 不过每次调用onNext都会去dispose前一个观察者。所以只有最后发射的一项数据的观察者能接受到数据。 



3. concatmap

   ```java
   void drain() {
               if (getAndIncrement() != 0) {
                   return;
               }
   
               for (;;) {
                   if (disposed) {
                       queue.clear();
                       return;
                   }
                   if (!active) {
   
                       boolean d = done;
   
                       T t;
   
                       try {
                           t = queue.poll();
                       } catch (Throwable ex) {
                           Exceptions.throwIfFatal(ex);
                           dispose();
                           queue.clear();
                           downstream.onError(ex);
                           return;
                       }
   
                       boolean empty = t == null;
   
                       if (d && empty) {
                           disposed = true;
                           downstream.onComplete();
                           return;
                       }
   
                       if (!empty) {
                           ObservableSource<? extends U> o;
   
                           try {
                               o = ObjectHelper.requireNonNull(mapper.apply(t), "The mapper returned a null ObservableSource");
                           } catch (Throwable ex) {
                               Exceptions.throwIfFatal(ex);
                               dispose();
                               queue.clear();
                               downstream.onError(ex);
                               return;
                           }
   
                           active = true;
                           o.subscribe(inner);
                       }
                   }
   
                   if (decrementAndGet() == 0) {
                       break;
                   }
               }
   ```

   Concatmap 每次上游调用onNext先把数据t存入queue中 再调用drain方法 queue.poll()获取t

   再调用mapper.apply(t) 生成新的ObservableSource去subscribe observer不过这里有个active的判断

   如果这个active时true也是还在走o.subscribe(inner)里面的方法 就不会走下一次循环.inner的onComplete

   调用后将active搞成fales 这样再走下次循环 所以concatmap是会按照数据源发射的顺序执行的

4. concat操作符用的也是ObservableConcatMap 与concatMap区别是 concatMap中的source就是调用concatMap的observable而concat是在参数中传入source的。