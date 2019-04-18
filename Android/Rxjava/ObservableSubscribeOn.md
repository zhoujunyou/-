### ObservableSubscribeOn

这个类比较简单

```java
public final class ObservableSubscribeOn<T> extends AbstractObservableWithUpstream<T, T> {
    final Scheduler scheduler;

    public ObservableSubscribeOn(ObservableSource<T> source, Scheduler scheduler) {
        super(source);
        this.scheduler = scheduler;
    }

    @Override
    public void subscribeActual(final Observer<? super T> s) {
        final SubscribeOnObserver<T> parent = new SubscribeOnObserver<T>(s);

        s.onSubscribe(parent);
		//这里将线程的dispose 设置到SubscribeOnObserver中 用于后面中断流的时候中端线程
        parent.setDisposable(scheduler.scheduleDirect(new SubscribeTask(parent)));
    }

    static final class SubscribeOnObserver<T> extends AtomicReference<Disposable> implements Observer<T>, Disposable {

        private static final long serialVersionUID = 8094547886072529208L;
        final Observer<? super T> actual;

        final AtomicReference<Disposable> s;

        SubscribeOnObserver(Observer<? super T> actual) {
            this.actual = actual;
            this.s = new AtomicReference<Disposable>();
        }

        @Override
        public void onSubscribe(Disposable s) {
            DisposableHelper.setOnce(this.s, s);
        }

        @Override
        public void onNext(T t) {
            actual.onNext(t);
        }

        @Override
        public void onError(Throwable t) {
            actual.onError(t);
        }

        @Override
        public void onComplete() {
            actual.onComplete();
        }

        @Override
        public void dispose() {
            //这个是在onSubscribe中设置的上游的Disposable
            DisposableHelper.dispose(s);
            //这个是线程的Disposabe
            DisposableHelper.dispose(this);
        }

        @Override
        public boolean isDisposed() {
            return DisposableHelper.isDisposed(get());
        }

        void setDisposable(Disposable d) {
            DisposableHelper.setOnce(this, d);
        }
    }

    final class SubscribeTask implements Runnable {
        private final SubscribeOnObserver<T> parent;

        SubscribeTask(SubscribeOnObserver<T> parent) {
            this.parent = parent;
        }

        @Override
        public void run() {
            //直接用SubscribeOnObserver订阅上游的 subcribe中执行的就会切换到新线程
            source.subscribe(parent);
        }
    }
}
```



总结：

subscribeOn会将订阅的逻辑切(Observable)到新线程 observeOn 会将观察的逻辑切到新线程

subscribeOn会一直向上游的Observable影响 而observeOn会一直向下游的Observer影响

我觉得可以这样理解.subscribeOn决定上游Observable subscribeActual方法的执行线程

observeOn决定下游Observer的onXxx方法的执行线程。反过来Observable subscribeActual方法线程

由下方最近的一个subscribeOn决定 没有subscribeOn则在当前调用线程执行。Observer的onXxx方法线程由上方最近的一个observeOn决定 没有observeOn则由上方最远即第一个subscribeOn决定 

