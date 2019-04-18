### LambdaObserver

```java
subscribe()
subscribe(Consumer<? super T> onNext)
subscribe(Consumer<? super T> onNext, Consumer<? super Throwable> onError)
subscribe(Consumer<? super T> onNext, Consumer<? super Throwable> onError,
            Action onComplete)
public final Disposable subscribe(Consumer<? super T> onNext, Consumer<? super Throwable> onError,
            Action onComplete, Consumer<? super Disposable> onSubscribe) {
        ObjectHelper.requireNonNull(onNext, "onNext is null");
        ObjectHelper.requireNonNull(onError, "onError is null");
        ObjectHelper.requireNonNull(onComplete, "onComplete is null");
        ObjectHelper.requireNonNull(onSubscribe, "onSubscribe is null");

        LambdaObserver<T> ls = new LambdaObserver<T>(onNext, onError, onComplete, onSubscribe);

        subscribe(ls);

        return ls;
    }   
    
```
上面Obeservable中的几个subscribe方法最终会调用subscribe(ls) 并返回ls

1. 其实LambdaObserver就是对onNext onError onComplete onSubscribe 和Disposable的一个包装

   最后都会调用到里面的方法

2. ```java
   @Override
       public void onSubscribe(Disposable s) {
           //这一行就是将上游的Disposable设置到LambdaObserver中 这也是我们可以用subscribe返回的disposable丢弃上游数据的原因
           if (DisposableHelper.setOnce(this, s)) {
               try {
                   onSubscribe.accept(this);
               } catch (Throwable ex) {
                   Exceptions.throwIfFatal(ex);
                   s.dispose();
                   onError(ex);
               }
           }
       }
   ```


