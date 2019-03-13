### ObservableZip

```java
  public void drain() {
            if (getAndIncrement() != 0) {
                return;
            }

            int missing = 1;

            final ZipObserver<T, R>[] zs = observers;
            final Observer<? super R> a = downstream;
            final T[] os = row;
            final boolean delayError = this.delayError;

            for (;;) {

                for (;;) {
                    int i = 0;
                    int emptyCount = 0;
                    for (ZipObserver<T, R> z : zs) {
                        if (os[i] == null) {
                            boolean d = z.done;
                            T v = z.queue.poll();
                            boolean empty = v == null;

                            if (checkTerminated(d, empty, a, delayError, z)) {
                                return;
                            }
                            if (!empty) {
                                os[i] = v;
                            } else {
                                emptyCount++;
                            }
                        } else {
                            if (z.done && !delayError) {
                                Throwable ex = z.error;
                                if (ex != null) {
                                    cancel();
                                    a.onError(ex);
                                    return;
                                }
                            }
                        }
                        i++;
                    }

                    if (emptyCount != 0) {
                        break;
                    }

                    R v;
                    try {
                        v = ObjectHelper.requireNonNull(zipper.apply(os.clone()), "The zipper returned a null value");
                    } catch (Throwable ex) {
                        Exceptions.throwIfFatal(ex);
                        cancel();
                        a.onError(ex);
                        return;
                    }

                    a.onNext(v);

                    Arrays.fill(os, null);
                }

                missing = addAndGet(-missing);
                if (missing == 0) {
                    return;
                }
            }
        }
```

1. 一个source subscribe一个ZipObserver。

2. 每次调用onXXX方法 执行drain() 调用onNext 方法ZipObserver.queue.offer(t)  

3. drain执行时会遍历observer 如果os[i]中没有t  执行queue.poll() 结果存入os[i] 

4. 从各个queue拉取一次结果后 os[] 如果存满则v=zipper.apply(os.clone() 调用downstream.onNext(v)

   否则结束内层循环等待下一次drain方法被调用

zip操作会讲每个source第一个数据通过zipper.apply(os.clone()）后发出 其中每个source调用onXXX方法都会执行drain() 一旦某个source 被终止了整个流也会被终止.