

### ObservableCreate

背景：rxjava一直在用 一直没深入想深入了解下rxjava 先从ObservableCreate如手把。

```java
@CheckReturnValue
    @SchedulerSupport(SchedulerSupport.NONE)
    public static <T> Observable<T> create(ObservableOnSubscribe<T> source) {
        ObjectHelper.requireNonNull(source, "source is null");
        return RxJavaPlugins.onAssembly(new ObservableCreate<T>(source));
    }
```

以上是创建ObservableCreate方法。
![img](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/ObservableCreate.png)
subscribe时序图



![img](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/ObservableCreateUML (1).png)
UML类图



##### 简要记录

1. ObservableSource 表示被观察对象可订阅
2. Observable  ObservableSource的抽象实现以及定义了一堆工厂方法
3. ObservableCreate Observable实现类主要实现subscribeActual方法
4. ObservableOnSubscribe 一个用于Observable被subscribe之后的调用ObservableEmitter的接口
5. Emitter 用于发射生产出的数据源的接口
6. ObservableEmitter Emitter的实现接口定义了额外的一些方法比如serialize(),tryOnError(),setDisposable()
   1. tryOnError用于判断下游是否取消了流或者流终止了 如果是直接RxJavaPlugins.onError(t)抛出错误

      所以这里要注意如果没对RxJavaPlugins.onError(t)处理  在主动调用onError前可以先判断是否isDisposeb避免调用dispose后崩溃

   2. serialize() 返回一个保证onNext 在onError或onComplete之前调用的ObservableEmitter这里就是SerializedEmitter
7. CreateEmitter ObservableEmitter具体实现类
8. Observer 观察者 用于接收数据源
9. Disposable 表示是一个可取消的源



#####SerializedEmitter是如何保证顺序呢

```java
  @Override
        public void onNext(T t) {
            if (emitter.isDisposed() || done) {
                return;
            }
            if (t == null) {
                onError(new NullPointerException("onNext called with null. Null values are generally not allowed in 2.x operators and sources."));
                return;
            }
            //并发时 第一项进入这个if
            if (get() == 0 && compareAndSet(0, 1)) {
                emitter.onNext(t);
                //上一行走完后-1 如果不为0则认为队列中缓存了数据走drainLoop()
                if (decrementAndGet() == 0) {
                    return;
                }
            } else {
                SimpleQueue<T> q = queue;
                synchronized (q) {
                    q.offer(t);
                }
                //如果第一个onNext没处理完+1直接return 处理完则走drainLoop()
                if (getAndIncrement() != 0) {
                    return;
                }
            }
            drainLoop();
        }

        @Override
        public void onError(Throwable t) {
            if (!tryOnError(t)) {
                RxJavaPlugins.onError(t);
            }
        }

        @Override
        public boolean tryOnError(Throwable t) {
            if (emitter.isDisposed() || done) {
                return false;
            }
            if (t == null) {
                t = new NullPointerException("onError called with null. Null values are generally not allowed in 2.x operators and sources.");
            }
            if (error.addThrowable(t)) {
                done = true;
                drain();
                return true;
            }
            return false;
        }

        @Override
        public void onComplete() {
            if (emitter.isDisposed() || done) {
                return;
            }
            done = true;
            drain();
        }
		//这个方法会在tryOnError onComplete 中调用
        void drain() {
            //+1为了表示调用了上面的方法 这样onNext就会走队列了  ==0判断用保证drainLoop没在执行 
            if (getAndIncrement() == 0) {
                drainLoop();
            }
        }

        void drainLoop() {
            ObservableEmitter<T> e = emitter;
            SpscLinkedArrayQueue<T> q = queue;
            AtomicThrowable error = this.error;
            int missed = 1;
            //调用drainLoop的线程会暂时会进入这个循环直到队列中数据处理完毕。
            for (;;) {

                for (;;) {
                    //流取消
                    if (e.isDisposed()) {
                        q.clear();
                        return;
                    }
					//调用了onError
                    if (error.get() != null) {
                        q.clear();
                        e.onError(error.terminate());
                        return;
                    }
					//是否调用了onComplete
                    boolean d = done;
                    T v = q.poll();

                    boolean empty = v == null;
                    if (d && empty) {
                        e.onComplete();
                        return;
                    }
					//队列为空先退到外层循环
                    if (empty) {
                        break;
                    }

                    e.onNext(v);
                }
			
                missed = addAndGet(-missed);
                //代表队列中已经没数据了。
                if (missed == 0) {
                    break;
                }
            }
        }
```



