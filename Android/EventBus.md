### EventBus

eventbus源码比较简单
1. **EventBusAnnotationProcessor**   AbstractProcessor的原理在glide中已经分析过了

   ```java
   //支持处理哪些注解，这里支持org.greenrobot.eventbus.Subscribe
   @SupportedAnnotationTypes("org.greenrobot.eventbus.Subscribe")
   //支持哪几个Options,Options的参数在build.gradle文件中配置
   @SupportedOptions(value = {"eventBusIndex", "verbose"})
   ```

   ```
   
           javaCompileOptions {
               annotationProcessorOptions {
    arguments = [eventBusIndex: 'org.greenrobot.eventbusperf.MyEventBusIndex']
               }
           }
   ```

   ```java
   String index = processingEnv.getOptions().get(OPTION_EVENT_BUS_INDEX);
   if (index == null) {
       messager.printMessage(Diagnostic.Kind.ERROR, "No option " + OPTION_EVENT_BUS_INDEX +
               " passed to annotation processor");
       return false;
   }
   ```

   这里可以看到不给配置eventBusIndex直接return了，那EventBus还是用的运行时反射，所以最好配置一下 ,编译时提前预置。再调用下EventBus.builder().addIndex(new MyEventBusIndex())。



2. register过程

   * 找到对应的SubscriberMethod列表， 先从METHOD_CACHE缓存中获取 如果有直接返回。然后走findUsingInfo如果EventBus添加了Index则从Index中获取subscriberInfo，这个Index就是我们编译时生成的类。最后还没获取到则走findUsingReflectionInSingleClass 通过clazz反射获取方法生成SubscriberMethod。SubscriberMethod包含method，threadMode，eventType，priority，sticky等信息
   * 将subscriber 与SubscriberMethod生成对应Subscription放入eventType对应的subscriptions列表，subscriptions按照priority排序，维护一个Map<Object, List<Class<?>>> typesBySubscriber用于unregister

   post过程

   * 获取当前线程的PostingThreadState postingState.eventQueue中加入event。根据eventClass获取subscriptions。遍历subscriptions调用postToSubscription 中根据threadMode使用不同的Poster执行线程调度，线程调度也很简单。

    sticky事件

   * postSticky 将event存入ConcurrentHashMap<Class<?>, Object> stickyEvents的一个
   * register如果发现SubscriberMethod的sticky为true从stickyEvents获取event调用postToSubscription

   unregiter

   * 从subscriptionsByEventType中移除，从typesBySubscriber中移除。

