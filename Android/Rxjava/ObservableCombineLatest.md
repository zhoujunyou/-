### ObservableCombineLatest

对应操作符号 ：combineLatest

```java
 LatestCoordinator(Observer<? super R> actual,
                Function<? super Object[], ? extends R> combiner,
                int count, int bufferSize, boolean delayError) {
            this.downstream = actual;
            this.combiner = combiner;
            this.delayError = delayError;
            this.latest = new Object[count];
            CombinerObserver<T, R>[] as = new CombinerObserver[count];
            for (int i = 0; i < count; i++) {
                as[i] = new CombinerObserver<T, R>(this, i);
            }
            this.observers = as;
            this.queue = new SpscLinkedArrayQueue<Object[]>(bufferSize);
        }

        public void subscribe(ObservableSource<? extends T>[] sources) {
            Observer<T>[] as = observers;
            int len = as.length;
            downstream.onSubscribe(this);
            for (int i = 0; i < len; i++) {
                if (done || cancelled) {
                    return;
                }
                sources[i].subscribe(as[i]);
            }
        }
```

```java
 @Override
        public void onNext(T t) {
            parent.innerNext(index, t);
        }
```

```java
 void innerNext(int index, T item) {
            boolean shouldDrain = false;
            synchronized (this) {
                Object[] latest = this.latest;
                if (latest == null) {
                    return;
                }
                Object o = latest[index];
                int a = active;
                if (o == null) {
                    active = ++a;
                }
                latest[index] = item;
                if (a == latest.length) {
                    queue.offer(latest.clone());
                    shouldDrain = true;
                }
            }
            if (shouldDrain) {
                drain();
            }
        }
```

drain()

```java
				Object[] s = q.poll();
                    boolean empty = s == null;

                    if (d && empty) {
                        clear(q);
                        Throwable ex = errors.terminate();
                        if (ex == null) {
                            a.onComplete();
                        } else {
                            a.onError(ex);
                        }
                        return;
                    }

                    if (empty) {
                        break;
                    }	
					R v;

                    try {
                        v = ObjectHelper.requireNonNull(combiner.apply(s), "The combiner returned a null value");
                    } catch (Throwable ex) {
                 		...
                        return;
                    }

                    a.onNext(v);
```



sources[i].subscribe(as[i]) 每个source subscibe对应的CombinerObserver  CombinerObserver的onNext调用到LatestCoordinator的innerNext()  上游source最近发射的数据保存到相应的latest数组中 直到latest数组存满 将latest提交到queue 调用drain().   从queue中poll出latest  再用combiner.apply转换得到v  发射v



总结：缓存每个源最近发射的数据，直到每个源都发射数据后 进行mapper转换后向下游发射数据.