 ### Single,Maybe,Completable的总结

个人觉得是为了更方便的调用API  还有一些性能上的提升。比如Single的设计本身就不支持并发这样就不用加个种锁程序性能随之就提升了。

Observable ,Flowable 可以发射多个item,Flowable支持背压

1. Single: 只支持发射一个item 或者抛出一个错误 onComplete()和onNext()被结合成了一个onSuccess  Android 网络请求就适合用这个 只需要收到一次请求结果或者是错误
2. Maybe: Maybe和Single比较相似 Maybe允许不发射数据有onComplete()方法
3. Completable 只关心任务执行完成或者失败 




