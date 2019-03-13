1. **doFinally，doAfterTerminate区别**

   doAfterTerminate Execute an action after an onError, onComplete

   doFinally Execute an action after an onError, onComplete or a dispose event 

   doFinally在被dispose时也会调用而doAfterTerminate不会

2. **doOnNext,doAfterNext区别**

  doAfterNext  Calls a consumer after pushing the current item to the downstream.

   doOnNext  invokes an action when it calls onNext

   doOnNext 在执行downstream.onNext前调用 doAfterNext在执行downstream.onNext后调用

---

3. **firstOrError**

   Returns a Single that emits only the very first item emitted by this Observable or

   signals a {@link NoSuchElementException} if this Observable is empty.

   返回第一个值 如果在onComplete前没有值抛出NoSuchElementException

---

4. **onErrorReturn,onErrorResumeNext**
  onErrorReturn ： onError被调用时 用Function<? super Throwable, ? extends T>将错误转换t  往下游发出t

  并调用onComplete()

   onErrorResumeNext: onError第一次被调用时 用Function<? super Throwable, ? extends ObservableSource<? extends T>>得到一个observable并subscribe当前observer。onError再次被调用时
  调用downstream.onError

5. **switchIfEmpty**

    如果source没有调用onNext 就调用onComplete 则用other 去subscribe  observer。

6. **hide**

    隐藏Observable中像Subject这种实现了其他接口的接口调用。仅对ObservableSource和

     Disposable进行了包装。

7. **startwith**

    startwith(other)->concatArray(other,this)->ObservableConcatMap 

    最后还是用ObservableConcatMap中挨个发出。
