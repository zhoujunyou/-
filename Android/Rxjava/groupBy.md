### groupBy

* 函数返回类型Observable<GroupedObservable<K, V>> 所以最后会向downstream发送GroupedObservable<K, V>  类型数据

* GroupByObserver<T, K, V> 中 onNext方法keySelector将T转换为K 
* 根据K第一次会创建GroupedObservable的子类GroupedUnicast 存入ConcurrentHashMap<Object, GroupedUnicast<K, V>>()中  向downstream发送GroupedUnicast
* valueSelector将T转为V
* 调用GroupedUnicast onNext(v)->state.onNext(v)->queue.offer(v)->drain()。
* 如果GroupedUnicast没被subscibe数据就会被缓存在queue中 直到被观察订阅。

 栗子

```java
Observable.range(0, 100)
                .groupBy(new Function<Integer, Integer>() {

                    @Override
                    public Integer apply(Integer i) {
                        return i % 2;
                    }
                })
                .flatMap(new Function<GroupedObservable<Integer, Integer>, Observable<Integer>>() {

                    @Override
                    public Observable<Integer> apply(GroupedObservable<Integer, Integer> group) {
                        if (group.getKey() == 0) {
                            return group.delay(100, TimeUnit.MILLISECONDS).map(new Function<Integer, Integer>() {
                                @Override
                                public Integer apply(Integer t) {
                                    return t * 10;
                                }

                            });
                        } else {
                            return group;
                        }
                    }
                })
                .subscribe(new DefaultObserver<Integer>() {

                    @Override
                    public void onComplete() {
                        System.out.println("=> onComplete");
                        latch.countDown();
                    }

                    @Override
                    public void onError(Throwable e) {
                        e.printStackTrace();
                        latch.countDown();
                    }

                    @Override
                    public void onNext(Integer s) {
                        eventCounter.incrementAndGet();
                        System.out.println("=> " + s);
                    }
                });
```

