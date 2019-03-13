 ### Subject系列

Subject 是一个继承了Observable 又实现了Observer的抽象类  一个源允许有多个观察者 

实现Subject的类有PublishSubject,BehaviorSubject,ReplaySubject,AsyncSubject 先分析这几个



####Subject共同点

1. 每个类都在subscribeActual时都有个add 和remove方法

```java
boolean add(ReplayDisposable<T> rs) {
        for (;;) {
            ReplayDisposable<T>[] a = observers.get();
            if (a == TERMINATED) {
                return false;
            }
            int len = a.length;
            @SuppressWarnings("unchecked")
            ReplayDisposable<T>[] b = new ReplayDisposable[len + 1];
            System.arraycopy(a, 0, b, 0, len);
            b[len] = rs;
            if (observers.compareAndSet(a, b)) {
                return true;
            }
        }
    }

    @SuppressWarnings("unchecked")
    void remove(ReplayDisposable<T> rs) {
        for (;;) {
            ReplayDisposable<T>[] a = observers.get();
            if (a == TERMINATED || a == EMPTY) {
                return;
            }
            int len = a.length;
            int j = -1;
            for (int i = 0; i < len; i++) {
                if (a[i] == rs) {
                    j = i;
                    break;
                }
            }

            if (j < 0) {
                return;
            }
            ReplayDisposable<T>[] b;
            if (len == 1) {
                b = EMPTY;
            } else {
                b = new ReplayDisposable[len - 1];
                System.arraycopy(a, 0, b, 0, j);
                System.arraycopy(a, j + 1, b, j, len - j - 1);
            }
            if (observers.compareAndSet(a, b)) {
                return;
            }
        }
    }
```

添加和删除都用拷贝数组的方式来实现 。 



#### PublicSubject

1. subscribe发生在onNext之后 observer才能够观察到数据。

2. 如果subject调用了onError,onComplete 后subscribe的observer只会观察到上一个error或者complete 已subscibe的observer 后面即使调了onNext也收不到数据

####BehaviorSubject

1. 和PublicSubject的区别 是observer被subscribe之后调用emitFirst()将会接收到最近一个事件 。

####AsyncSubject

1. 调用onComplete才会向所有observer发射最近一个数据  之后不在发射。onNext可发生在subscribe之前或之后.

#### ReplaySubject

1. 每次调用onNext会将数据加入一个buffer中  并调用buffer.replay 会将之前缓存的数据一起发射给observer

2. buffer有三种模式

   1.  UnboundedReplayBuffer 无限制 内部用Arraylist实现

   2.  SizeBoundReplayBuffer 限制大小

      ![image-20181027182152934](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20181027182152934.png)

       addFinal 会将一个notificationLite添加到尾部 再调用trimHead

      当head.vaule不为空时 新建一个value为null的Node作为head 用于避免内存泄漏

       replay时循环调用head.get() 发射数据到observer。

   3.  SizeAndTimeBoundReplayBuffer 限制时间和大小

       对应的是TimeNode 对于Node多了个time字段 onNext时存入time字段 发射给observer时

      计算当前时间与maxAge时间差是否大于time字段 用来限制时间。
