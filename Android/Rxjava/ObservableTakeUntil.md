### ObservableTakeUntil

对应操作符: takeUntil

```java
 @Override
    public void subscribeActual(Observer<? super T> child) {
        TakeUntilMainObserver<T, U> parent = new TakeUntilMainObserver<T, U>(child);
        child.onSubscribe(parent);

        other.subscribe(parent.otherObserver);
        source.subscribe(parent);
    }
```

```java
 @Override
            public void onNext(U t) {
                DisposableHelper.dispose(this);
                otherComplete();
            }
```

```java
 void otherComplete() {
            DisposableHelper.dispose(upstream);
            HalfSerializer.onComplete(downstream, this, error);
        }
```

实现比较简单 other源一旦调用onNext->otherComplete->DisposableHelper.dispose(upstream)  最终取消上游数据源.