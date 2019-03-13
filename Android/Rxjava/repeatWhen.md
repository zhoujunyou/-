### repeatWhen

```java


 other = ObjectHelper.requireNonNull(handler.apply(signaller), "The handler returned a null ObservableSource");

        other.subscribe(parent.inner);
```

```java
final class InnerRepeatObserver extends AtomicReference<Disposable> implements Observer<Object> {

    private static final long serialVersionUID = 3254781284376480842L;

    @Override
    public void onSubscribe(Disposable d) {
        DisposableHelper.setOnce(this, d);
    }

    @Override
    public void onNext(Object t) {
        innerNext();
    }

    @Override
    public void onError(Throwable e) {
        innerError(e);
    }

    @Override
    public void onComplete() {
        innerComplete();
    }
}
```

```java
void subscribeNext() {
    if (wip.getAndIncrement() == 0) {

        do {
            if (isDisposed()) {
                return;
            }

            if (!active) {
                active = true;
                source.subscribe(this);
            }
        } while (wip.decrementAndGet() != 0);
    }
}

 void innerError(Throwable ex) {
            DisposableHelper.dispose(upstream);
            HalfSerializer.onError(downstream, ex, this, error);
        }

        void innerComplete() {
            DisposableHelper.dispose(upstream);
            HalfSerializer.onComplete(downstream, this, error);
        }
```

我们传入的other相当于一个信号，给other不同的状态会走到InnerRepeatObserver的onXXX方法中。

innerError，innerComplete会终止上游执行。而innerNext会走到subscribeNext。如果上个事件已经complete

则会重复订阅。

