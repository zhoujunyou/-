### RxLifeCycle

用法

```java
compose(RxLifecycleAndroid.bindActivity(lifecycleSubject))
```

```java
public abstract class BaseActivity extends AppCompatActivity{
    protected final BehaviorSubject<ActivityEvent> lifecycleSubject = BehaviorSubject.create();
     protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        lifecycleSubject.onNext(ActivityEvent.CREATE);
    }
    
     protected void onDestroy() {
        super.onDestroy();
        lifecycleSubject.onNext(ActivityEvent.DESTROY);
    }
}
```



RxLifecycleAndroid.bindActivity()

```java
public static <T> LifecycleTransformer<T> bindActivity(@NonNull final Observable<ActivityEvent> lifecycle) {
        return bind(lifecycle, ACTIVITY_LIFECYCLE);
    }
```



lifecycle为 BaseActivity中的lifecycleSubject 会在activity 调用生命周期函数时调用相应的ActivityEvent

而且是一个BehaviorSubject被订阅会先发送最近一次数据。

 ACTIVITY_LIFECYCLE 是一个Function用于ActivityEvent的对应关系 比如create转destroy  start转stop



```java
public static <T, R> LifecycleTransformer<T> bind(@Nonnull Observable<R> lifecycle,
                                                      @Nonnull final Function<R, R> correspondingEvents) {
        checkNotNull(lifecycle, "lifecycle == null");
        checkNotNull(correspondingEvents, "correspondingEvents == null");
        return bind(takeUntilCorrespondingEvent(lifecycle.share(), correspondingEvents));
    }

    private static <R> Observable<Boolean> takeUntilCorrespondingEvent(final Observable<R> lifecycle,
                                                                       final Function<R, R> correspondingEvents) {
        return Observable.combineLatest(
            lifecycle.take(1).map(correspondingEvents),
            lifecycle.skip(1),
            new BiFunction<R, R, Boolean>() {
                @Override
                public Boolean apply(R bindUntilEvent, R lifecycleEvent) throws Exception {
                    return lifecycleEvent.equals(bindUntilEvent);
                }
            })
            .onErrorReturn(Functions.RESUME_FUNCTION)
            .filter(Functions.SHOULD_COMPLETE);
    }
```

lifecycle.share() 功能用于直到有一个observe被subscribe才会发射数据。这样也就可以知道订阅时候Activity的生命周期了。 

lifecycle.take(1).map(correspondingEvents)  用于得到订阅时候Activity的生命周期对应结束的生命周期事件的源

lifecycle.skip(1) 跳过第一个事件 即上面一个事件 的源。

combineLatest  缓存上面两个源最新的数据 当lifecycle.skip(1)取到的值等于结束事件时 返回true 

filter过滤了false只保留了true事件



LifecycleTransformer

```java
 @Override
    public ObservableSource<T> apply(Observable<T> upstream) {
        return upstream.takeUntil(observable);
    }
```

这里的observable就是上面的到的observable 

takeUntil的效果是 当observable 接受到上游数据时 就会调用upstream.dispose()



综上所述，当activity走到订阅生命周期相对应结束的生命周期时就会触发dispose上游源.







