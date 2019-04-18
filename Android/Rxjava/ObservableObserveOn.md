### ObservableObserveOn

```java
observable
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(i->{Log.i("rxjava",i)})
```

上面切换到主线程的方法经常用 稍微看一下 

ObservableObserveOn里知识点还是比较多 从简单开始分析

1. 为何观察者中的方法能切到指定线程
2. rxjava 中的线程调度实现 （io.reactivex.internal.schedulers包下）之后再看看
3. QueueDisposable 中的融合模式



1. 将onXxx方法切到指定线程的原理
2. ![image-20181018161503023](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20181018161503023.png)

```java
 static final class ObserveOnObserver<T> extends BasicIntQueueDisposable<T>
    implements Observer<T>, Runnable {

        private static final long serialVersionUID = 6576896619930983584L;
        final Observer<? super T> actual;
        final Scheduler.Worker worker;
        final boolean delayError;
        final int bufferSize;

        SimpleQueue<T> queue;

        Disposable s;

        Throwable error;
        volatile boolean done;

        volatile boolean cancelled;

        int sourceMode;

        boolean outputFused;

        ObserveOnObserver(Observer<? super T> actual, Scheduler.Worker worker, boolean delayError, int bufferSize) {
            this.actual = actual;
            this.worker = worker;
            this.delayError = delayError;
            this.bufferSize = bufferSize;
        }

        @Override
        public void onSubscribe(Disposable s) {
            if (DisposableHelper.validate(this.s, s)) {
                this.s = s;
                if (s instanceof QueueDisposable) {
                    @SuppressWarnings("unchecked")
                    QueueDisposable<T> qd = (QueueDisposable<T>) s;

                    int m = qd.requestFusion(QueueDisposable.ANY | QueueDisposable.BOUNDARY);

                    if (m == QueueDisposable.SYNC) {
                        sourceMode = m;
                        queue = qd;
                        done = true;
                        actual.onSubscribe(this);
                        schedule();
                        return;
                    }
                    if (m == QueueDisposable.ASYNC) {
                        sourceMode = m;
                        queue = qd;
                        actual.onSubscribe(this);
                        return;
                    }
                }

                queue = new SpscLinkedArrayQueue<T>(bufferSize);

                actual.onSubscribe(this);
            }
        }

        @Override
        public void onNext(T t) {
            if (done) {
                return;
            }

            if (sourceMode != QueueDisposable.ASYNC) {
                queue.offer(t);
            }
            schedule();
        }

        @Override
        public void onError(Throwable t) {
            if (done) {
                RxJavaPlugins.onError(t);
                return;
            }
            error = t;
            done = true;
            schedule();
        }

        @Override
        public void onComplete() {
            if (done) {
                return;
            }
            done = true;
            schedule();
        }

        @Override
        public void dispose() {
            if (!cancelled) {
                cancelled = true;
                s.dispose();
                worker.dispose();
                if (getAndIncrement() == 0) {
                    queue.clear();
                }
            }
        }

        @Override
        public boolean isDisposed() {
            return cancelled;
        }

        void schedule() {
        	//这里和drainNormal的missed配合保证只有一个任务在执行
            if (getAndIncrement() == 0) {
                worker.schedule(this);
            }
        }
		//配合done和队列 响应上游的onXxx方法
        void drainNormal() {
            int missed = 1;

            final SimpleQueue<T> q = queue;
            final Observer<? super T> a = actual;

            for (;;) {
                if (checkTerminated(done, q.isEmpty(), a)) {
                    return;
                }

                for (;;) {
                    boolean d = done;
                    T v;

                    try {
                        v = q.poll();
                    } catch (Throwable ex) {
                        Exceptions.throwIfFatal(ex);
                        s.dispose();
                        q.clear();
                        a.onError(ex);
                        worker.dispose();
                        return;
                    }
                    boolean empty = v == null;

                    if (checkTerminated(d, empty, a)) {
                        return;
                    }

                    if (empty) {
                        break;
                    }

                    a.onNext(v);
                }

                missed = addAndGet(-missed);
                if (missed == 0) {
                    break;
                }
            }
        }

        void drainFused() {
            int missed = 1;

            for (;;) {
                if (cancelled) {
                    return;
                }

                boolean d = done;
                Throwable ex = error;

                if (!delayError && d && ex != null) {
                    actual.onError(error);
                    worker.dispose();
                    return;
                }

                actual.onNext(null);

                if (d) {
                    ex = error;
                    if (ex != null) {
                        actual.onError(ex);
                    } else {
                        actual.onComplete();
                    }
                    worker.dispose();
                    return;
                }

                missed = addAndGet(-missed);
                if (missed == 0) {
                    break;
                }
            }
        }

        @Override
        public void run() {
        	//可以和上游融合
            if (outputFused) {
                drainFused();
            } else {
                drainNormal();
            }
        }
		//这个方法主要用于流结束后 中断worker 
        boolean checkTerminated(boolean d, boolean empty, Observer<? super T> a) {
            if (cancelled) {
                queue.clear();
                return true;
            }
            if (d) {
                Throwable e = error;
                if (delayError) {
                    if (empty) {
                        if (e != null) {
                            a.onError(e);
                        } else {
                            a.onComplete();
                        }
                        worker.dispose();
                        return true;
                    }
                } else {
                    if (e != null) {
                        queue.clear();
                        a.onError(e);
                        worker.dispose();
                        return true;
                    } else
                    if (empty) {
                        a.onComplete();
                        worker.dispose();
                        return true;
                    }
                }
            }
            return false;
        }

        @Override
        public int requestFusion(int mode) {
            if ((mode & ASYNC) != 0) {
                outputFused = true;
                return ASYNC;
            }
            return NONE;
        }

        @Nullable
        @Override
        public T poll() throws Exception {
            return queue.poll();
        }

        @Override
        public void clear() {
            queue.clear();
        }

        @Override
        public boolean isEmpty() {
            return queue.isEmpty();
        }
    }
```

