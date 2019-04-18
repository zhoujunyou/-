### notifyDataSetChanged

 notifyDataSetChanged和notifyItemChanged 区别

*notifyDataSetChanged* 调用到RecyclerViewDataObserver.onChanged()

```java
@Override
public void onChanged() {
    assertNotInLayoutOrScroll(null);
    mState.mStructureChanged = true;

    setDataSetChangedAfterLayout();
    if (!mAdapterHelper.hasPendingUpdates()) {
        requestLayout();
    }
}
```

```java
/**
 * Set to true when an adapter data set changed notification is received.
 * In that case, we cannot run any animations since we don't know what happened until layout.
 *
 * Attached items are invalid until next layout, at which point layout will animate/replace
 * items as necessary, building up content from the (effectively) new adapter from scratch.
 *
 * Cached items must be discarded when setting this to true, so that the cache may be freely
 * used by prefetching until the next layout occurs.
 *
 * @see #setDataSetChangedAfterLayout()
 */
boolean mDataSetHasChangedAfterLayout = false;
```

上面这个值会在adapter数据改变时设置为true ，dispatchLayoutStep3后设置为false 



->RecyclerView.markKnownViewsInvalid() 所有子view的holder会添加

 holder.addFlags(ViewHolder.FLAG_UPDATE | ViewHolder.FLAG_INVALID);



requestLayout被调用后走到LinearLayoutManager onLayoutChildren->RecyclerView.scrapOrRecycleView 中viewHolder.isInvalid()成立

 recycleViewHolderInternal->putRecycledView ->ViewHolder.resetInternal()

这里将ViewHolder的mFlags置为0. 后续走到tryGetViewHolderForPositionByDeadline方法中

 !holder.isBound()成立后重新走bindViewHolder方法。所以notifyDataSetChanged会将可见的子view重新bind一次。

*notifyItemChanged*  调用到 RecyclerViewDataObserve.onItemRangeChanged

 AdapterHelper

```java
boolean onItemRangeChanged(int positionStart, int itemCount, Object payload) {
    if (itemCount < 1) {
        return false;
    }
    mPendingUpdates.add(obtainUpdateOp(UpdateOp.UPDATE, positionStart, itemCount, payload));
    mExistingUpdateTypes |= UpdateOp.UPDATE;
    return mPendingUpdates.size() == 1;
}
```

调用requestLayout后

在processAdapterUpdatesAndSetAnimationFlags方法中AdpterHelper.preProcess,applyUpdate。调用Rv.findViewHolderForPosition找到当前position的viewHolder,AdpterHelper.postponeAndUpdateViewHolders

   范围内的holder  执行         holder.addFlags(ViewHolder.FLAG_UPDATE);

```java
void viewRangeUpdate(int positionStart, int itemCount, Object payload) {
    final int childCount = mChildHelper.getUnfilteredChildCount();
    final int positionEnd = positionStart + itemCount;

    for (int i = 0; i < childCount; i++) {
        final View child = mChildHelper.getUnfilteredChildAt(i);
        final ViewHolder holder = getChildViewHolderInt(child);
        if (holder == null || holder.shouldIgnore()) {
            continue;
        }
        if (holder.mPosition >= positionStart && holder.mPosition < positionEnd) {
            // We re-bind these view holders after pre-processing is complete so that
            // ViewHolders have their final positions assigned.
            holder.addFlags(ViewHolder.FLAG_UPDATE);
            holder.addChangePayload(payload);
            // lp cannot be null since we get ViewHolder from it.
            ((LayoutParams) child.getLayoutParams()).mInsetsDirty = true;
        }
    }
    mRecycler.viewRangeUpdate(positionStart, itemCount);
}
```

后续走到tryGetViewHolderForPositionByDeadline方法中holder.needsUpdate()成立重新走

bindViewHolder方法 所以notifyItemChanged只会去重新bind指定的view .

*连续调用多次notifyItemChanged* 每次调用会

`mPendingUpdates.add(obtainUpdateOp(UpdateOp.UPDATE, positionStart, itemCount, payload));`第一次回去调用requestLayout 。在preProcess中统一处理mPendingUpdates

holder.addFlags(ViewHolder.FLAG_UPDATE);



```java
void preProcess() {
    mOpReorderer.reorderOps(mPendingUpdates);
    final int count = mPendingUpdates.size();
    for (int i = 0; i < count; i++) {
        UpdateOp op = mPendingUpdates.get(i);
        switch (op.cmd) {
            case UpdateOp.ADD:
                applyAdd(op);
                break;
            case UpdateOp.REMOVE:
                applyRemove(op);
                break;
            case UpdateOp.UPDATE:
                applyUpdate(op);
                break;
            case UpdateOp.MOVE:
                applyMove(op);
                break;
        }
        if (mOnItemProcessedCallback != null) {
            mOnItemProcessedCallback.run();
        }
    }
    mPendingUpdates.clear();
}
```



后续流程和上面一样。

---
最近有个日历加流水的需求，日历可能本身就不太适合用RecyclerView写，因为是全部展示在界面上的。自己偷懒用RecyclerView写了 发现点击反应比较慢。排除了点击事件分发问题，发现每个子view bind一下还是要花个5-6ms。

![image-20190409175715308](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20190409175715308.png)

一个android.support.v7.widget.GridLayoutManager.layoutChunk耗时132ms 简直惨

![image-20190409175936549](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20190409175936549.png)
优化后耗时需要变的耗时17ms，不需要改变的6ms。看来rv还是不适合这种场景


![image-20190408215523156](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20190408215523156.png)

