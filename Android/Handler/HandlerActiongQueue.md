### HandlerActiongQueue

描述：Class used to enqueue pending work from Views when no Handler is attached.

用来保存handler还没添加到view上 提交的任务

**都知道View.post(Runable runable)方法中runable 中能使用view.getHeight()获取到height，在这里简单分析下原因**



![handlerActionQueue](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/handlerActionQueue.png)



```java
 public boolean post(Runnable action) {
        final AttachInfo attachInfo = mAttachInfo;
    
        if (attachInfo != null) {
            return attachInfo.mHandler.post(action);
        }

        // Postpone the runnable until we know on which thread it needs to run.
        // Assume that the runnable will be successfully placed after attach.
        getRunQueue().post(action);
        return true;
    }
```


如果attachInfo！=null(完成了`View.dispatchAttachedToWindow`)直接用attachInfo中的handler提交任务 否则先存入HandlerActiongQueue队列中

疑问点 为啥 performLayout()在dispatchAttachedToWindow之后却比View.post(runable)中的runable先执行？

因为Android是事件驱动的View.post(runable)是后添加到handler的performLayout本身是在一个Message中处理 处理完了handler才分发下一个Message.





```java
void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled = true;
            //添加一个同步Barrier
            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
				...
        }
    }		


void doTraversal() {
    		...
            performTraversals();
			...
        }
        
final class TraversalRunnable implements Runnable {
        @Override
        public void run() {
            doTraversal();
        }
    }

private void performTraversals() {
    ...
    host.dispatchAttachedToWindow(mAttachInfo, 0);
    performMeasure();
    performLayout(lp, mWidth, mHeight);
    performDraw();
    ...
    
}
```

