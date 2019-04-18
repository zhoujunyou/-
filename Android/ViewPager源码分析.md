### ViewPager源码分析

viewpager是一个能横向滑动切换子View的ViewGroup 从两个方面分析 1.如何布局 2如何响应滑动事件 

1. 如何布局 一个view的布局通过measure 测量长宽，layout决定位置，draw用于绘制到屏幕上。

   onMeasure暂不考虑子view是ViewPager.DecorView的情况  看populate方法。

    mCurItem：当前展示项

   ```java
    void populate(int newCurrentItem) {
        	//newCurrentItem 当前要展示项
           ItemInfo oldCurInfo = null;
           if (mCurItem != newCurrentItem) {
               //从mItems寻找
               oldCurInfo = infoForPosition(mCurItem);
               mCurItem = newCurrentItem;
           }
        	//PagerAdapter 一起看
           mAdapter.startUpdate(this);
   		//左右两边限制页数 超过限制的position会被移除 后面看怎么移除
           final int pageLimit = mOffscreenPageLimit;
           final int startPos = Math.max(0, mCurItem - pageLimit);
           final int N = mAdapter.getCount();
           final int endPos = Math.min(N - 1, mCurItem + pageLimit);
           int curIndex = -1;
           ItemInfo curItem = null;
           for (curIndex = 0; curIndex < mItems.size(); curIndex++) {
               final ItemInfo ii = mItems.get(curIndex);
               if (ii.position >= mCurItem) {
                   if (ii.position == mCurItem) curItem = ii;
                   break;
               }
           }
   
           if (curItem == null && N > 0) {
               //通过pagerAdapter添加新的item  添加到mItems中
               curItem = addNewItem(mCurItem, curIndex);
           }
   
           // Fill 3x the available width or up to the number of offscreen
           // pages requested to either side, whichever is larger.
           // If we have no current item we have no work to do.
           if (curItem != null) {
               float extraWidthLeft = 0.f;
               int itemIndex = curIndex - 1;
               ItemInfo ii = itemIndex >= 0 ? mItems.get(itemIndex) : null;
               final int clientWidth = getClientWidth();
               final float leftWidthNeeded = clientWidth <= 0 ? 0 :
                       2.f - curItem.widthFactor + (float) getPaddingLeft() / (float) clientWidth;
               //widthFactor为一个页面宽度百分比 viewpager 调用child的measure layout时会使用这个百分比
               //可更改PagerAdapter的 getPageWidth返回值查看效果
               //这个for的逻辑是从当前向左寻找 当前左边子view宽度+上当前view宽度>=2倍屏宽 且当前页下标超出了左边限制pageLimit 会进行remove和destroyItem操作。当前pos位置合理但没有Iteminfo调用addNewItem添加
               for (int pos = mCurItem - 1; pos >= 0; pos--) {
                   if (extraWidthLeft >= leftWidthNeeded && pos < startPos) {
                       if (ii == null) {
                           break;
                       }
                       if (pos == ii.position && !ii.scrolling) {
                           mItems.remove(itemIndex);
                           mAdapter.destroyItem(this, pos, ii.object);
                           if (DEBUG) {
                               Log.i(TAG, "populate() - destroyItem() with pos: " + pos
                                       + " view: " + ((View) ii.object));
                           }
                           itemIndex--;
                           curIndex--;
                           ii = itemIndex >= 0 ? mItems.get(itemIndex) : null;
                       }
                   } else if (ii != null && pos == ii.position) {
                       extraWidthLeft += ii.widthFactor;
                       itemIndex--;
                       ii = itemIndex >= 0 ? mItems.get(itemIndex) : null;
                   } else {
                       ii = addNewItem(pos, itemIndex + 1);
                       extraWidthLeft += ii.widthFactor;
                       curIndex++;
                       ii = itemIndex >= 0 ? mItems.get(itemIndex) : null;
                   }
               }
   
               float extraWidthRight = curItem.widthFactor;
               itemIndex = curIndex + 1;
               //右边 逻辑同上
               if (extraWidthRight < 2.f) {
                   ii = itemIndex < mItems.size() ? mItems.get(itemIndex) : null;
                   final float rightWidthNeeded = clientWidth <= 0 ? 0 :
                           (float) getPaddingRight() / (float) clientWidth + 2.f;
                   for (int pos = mCurItem + 1; pos < N; pos++) {
                       if (extraWidthRight >= rightWidthNeeded && pos > endPos) {
                           if (ii == null) {
                               break;
                           }
                           if (pos == ii.position && !ii.scrolling) {
                               mItems.remove(itemIndex);
                               mAdapter.destroyItem(this, pos, ii.object);
                               if (DEBUG) {
                                   Log.i(TAG, "populate() - destroyItem() with pos: " + pos
                                           + " view: " + ((View) ii.object));
                               }
                               ii = itemIndex < mItems.size() ? mItems.get(itemIndex) : null;
                           }
                       } else if (ii != null && pos == ii.position) {
                           extraWidthRight += ii.widthFactor;
                           itemIndex++;
                           ii = itemIndex < mItems.size() ? mItems.get(itemIndex) : null;
                       } else {
                           ii = addNewItem(pos, itemIndex);
                           itemIndex++;
                           extraWidthRight += ii.widthFactor;
                           ii = itemIndex < mItems.size() ? mItems.get(itemIndex) : null;
                       }
                   }
               }
   			//用于计算每页距离当前页面的offset  offset*width就是偏移距离保存在itemInfo中
               //在onLayout中从左到右摆放child 
               calculatePageOffsets(curItem, curIndex, oldCurInfo);
           }
   
           if (DEBUG) {
               Log.i(TAG, "Current page list:");
               for (int i = 0; i < mItems.size(); i++) {
                   Log.i(TAG, "#" + i + ": page " + mItems.get(i).position);
               }
           }
   		
        	//FragmentPagerAdapter 设置mCurrentPrimaryItem 用于标记fragment是否用户可见 
           mAdapter.setPrimaryItem(this, mCurItem, curItem != null ? curItem.object : null);
   		//FragmentPagerAdapter 中用于提交fragment事务 
           mAdapter.finishUpdate(this);
   
           // Check width measurement of current pages and drawing sort order.
           // Update LayoutParams as needed.
           final int childCount = getChildCount();
        //设置页面lp参数
           for (int i = 0; i < childCount; i++) {
               final View child = getChildAt(i);
               final LayoutParams lp = (LayoutParams) child.getLayoutParams();
               lp.childIndex = i;
               if (!lp.isDecor && lp.widthFactor == 0.f) {
                   // 0 means requery the adapter for this, it doesn't have a valid width.
                   final ItemInfo ii = infoForChild(child);
                   if (ii != null) {
                       lp.widthFactor = ii.widthFactor;
                       lp.position = ii.position;
                   }
               }
           }
        //设置子view绘制顺序 默认先绘制decor 再pos从小绘制
           sortChildDrawingOrder();
   
           if (hasFocus()) {
               View currentFocused = findFocus();
               ItemInfo ii = currentFocused != null ? infoForAnyChild(currentFocused) : null;
               if (ii == null || ii.position != mCurItem) {
                   for (int i = 0; i < getChildCount(); i++) {
                       View child = getChildAt(i);
                       ii = infoForChild(child);
                       if (ii != null && ii.position == mCurItem) {
                           if (child.requestFocus(View.FOCUS_FORWARD)) {
                               break;
                           }
                       }
                   }
               }
           }
       }
   
       
   ```

   ```java
   ItemInfo addNewItem(int position, int index) {
       ItemInfo ii = new ItemInfo();
       ii.position = position;
       //通过这个instantiateItem方法将view添加到ViewPager 
       ii.object = mAdapter.instantiateItem(this, position);
       ii.widthFactor = mAdapter.getPageWidth(position);
       if (index < 0 || index >= mItems.size()) {
           mItems.add(ii);
       } else {
           mItems.add(index, ii);
       }
       return ii;
   }
   ```

​	PagerAdapter destroyItem 用于回收视图 通常是调用ViewContainer.removeView.

​	**FragmentPagerAdapter和FragmentStatePagerAdapter  区别**

​	FragmentPagerAdapter destroyItem仅调用detach方法，detach后fragment还会存在FragmentManager			  的mActive中 所以在instantiateItem方法中会先尝试在mActive中寻找 找到直接attach否则再调用getItem(position)重新add。

FragmentStatePagerAdapter  destroyItem 先将当前frangment状态存到mSavedState中 将mFragments相应位置为null 调用FragmentManager removeFragment。在instantiateItem方法中先从mFragments中寻找 找到直接返回，找不到调用getItem(position)得到fragment 如果之前保存过该位置状态 将bundle赋值给fragment 加入mFragments 并add 。 FragmentStatePagerAdapter相比FragmentPagerAdapter多了 saveState和restoreState  

destroyItem生命周期 FragmentPagerAdapter 中fragment 走到onDestroryView而FragmentPagerAdapter中fragment走到onDetach



---



onLayout和onDraw

按照测量好的距离摆放页面  当前ViewPager



---

 onInterceptTouchEvent

Viewpager重写了onInterceptTouchEvent方法 该方法优先于onTouchEvent  并决定是否拦截触摸事件

```java
@Override
public boolean onInterceptTouchEvent(MotionEvent ev) {
    final int action = ev.getAction() & MotionEventCompat.ACTION_MASK;
    //cacel 和up让子view处理
    if (action == MotionEvent.ACTION_CANCEL || action == MotionEvent.ACTION_UP) {
        if (DEBUG) Log.v(TAG, "Intercept done!");
        resetTouch();
        return false;
    }
    if (action != MotionEvent.ACTION_DOWN) {
        //正在被拖拽
        if (mIsBeingDragged) {
            if (DEBUG) Log.v(TAG, "Intercept returning true!");
            return true;
        }
        if (mIsUnableToDrag) {
            if (DEBUG) Log.v(TAG, "Intercept returning false!");
            return false;
        }
    }

    switch (action) {
        case MotionEvent.ACTION_MOVE: {
            final int activePointerId = mActivePointerId;
            if (activePointerId == INVALID_POINTER) {
                break;
            }

            final int pointerIndex = ev.findPointerIndex(activePointerId);
            final float x = ev.getX(pointerIndex);
            final float dx = x - mLastMotionX;
            final float xDiff = Math.abs(dx);
            final float y = ev.getY(pointerIndex);
            final float yDiff = Math.abs(y - mInitialMotionY);
            if (DEBUG) Log.v(TAG, "Moved x to " + x + "," + y + " diff=" + xDiff + "," + yDiff);

            if (dx != 0 && !isGutterDrag(mLastMotionX, dx)
                    && canScroll(this, false, (int) dx, (int) x, (int) y)) {
                // Nested view has scrollable area under this point. Let it be handled there.
                mLastMotionX = x;
                mLastMotionY = y;
                mIsUnableToDrag = true;
                return false;
            }
            // x偏移大于slop且是y的两倍
            if (xDiff > mTouchSlop && xDiff * 0.5f > yDiff) {
                if (DEBUG) Log.v(TAG, "Starting drag!");
                mIsBeingDragged = true;
                // 让父view不拦截事件
                requestParentDisallowInterceptTouchEvent(true);
                setScrollState(SCROLL_STATE_DRAGGING);
                mLastMotionX = dx > 0
                        ? mInitialMotionX + mTouchSlop : mInitialMotionX - mTouchSlop;
                mLastMotionY = y;
                setScrollingCacheEnabled(true);
            } else if (yDiff > mTouchSlop) {
                // The finger has moved enough in the vertical
                // direction to be counted as a drag...  abort
                // any attempt to drag horizontally, to work correctly
                // with children that have scrolling containers.
                if (DEBUG) Log.v(TAG, "Starting unable to drag!");
                mIsUnableToDrag = true;
            }
            if (mIsBeingDragged) {
                //滑动viewpager
                if (performDrag(x)) {
                    ViewCompat.postInvalidateOnAnimation(this);
                }
            }
            break;
        }

        case MotionEvent.ACTION_DOWN: {
            /*
             * Remember location of down touch.
             * ACTION_DOWN always refers to pointer index 0.
             */
            mLastMotionX = mInitialMotionX = ev.getX();
            mLastMotionY = mInitialMotionY = ev.getY();
            mActivePointerId = ev.getPointerId(0);
            mIsUnableToDrag = false;

            mIsScrollStarted = true;
            mScroller.computeScrollOffset();
            if (mScrollState == SCROLL_STATE_SETTLING
                    && Math.abs(mScroller.getFinalX() - mScroller.getCurrX()) > mCloseEnough) {
                // Let the user 'catch' the pager as it animates.
                mScroller.abortAnimation();
                mPopulatePending = false;
                populate();
                mIsBeingDragged = true;
                requestParentDisallowInterceptTouchEvent(true);
                setScrollState(SCROLL_STATE_DRAGGING);
            } else {
                completeScroll(false);
                mIsBeingDragged = false;
            }

            if (DEBUG) {
                Log.v(TAG, "Down at " + mLastMotionX + "," + mLastMotionY
                        + " mIsBeingDragged=" + mIsBeingDragged
                        + "mIsUnableToDrag=" + mIsUnableToDrag);
            }
            break;
        }

        case MotionEventCompat.ACTION_POINTER_UP:
            onSecondaryPointerUp(ev);
            break;
    }

    if (mVelocityTracker == null) {
        mVelocityTracker = VelocityTracker.obtain();
    }
    mVelocityTracker.addMovement(ev);

    /*
     * The only time we want to intercept motion events is if we are in the
     * drag mode.
     */
    return mIsBeingDragged;
}
```

onInterceptTouchEvent方法主要功能拦截了偏向竖直方向的滑动，确定是否正在被拖拽。以及跟着手指滑动ViewPager 。通过是否被拖拽来拦截。



 onTouchEvent



```java
public boolean onTouchEvent(MotionEvent ev) {
        if (mFakeDragging) {
            // A fake drag is in progress already, ignore this real one
            // but still eat the touch events.
            // (It is likely that the user is multi-touching the screen.)
            return true;
        }

        if (ev.getAction() == MotionEvent.ACTION_DOWN && ev.getEdgeFlags() != 0) {
            // Don't handle edge touches immediately -- they may actually belong to one of our
            // descendants.
            return false;
        }

        if (mAdapter == null || mAdapter.getCount() == 0) {
            // Nothing to present or scroll; nothing to touch.
            return false;
        }

        if (mVelocityTracker == null) {
            mVelocityTracker = VelocityTracker.obtain();
        }
        mVelocityTracker.addMovement(ev);

        final int action = ev.getAction();
        boolean needsInvalidate = false;

        switch (action & MotionEventCompat.ACTION_MASK) {
            case MotionEvent.ACTION_DOWN: {
                mScroller.abortAnimation();
                mPopulatePending = false;
                populate();

                mLastMotionX = mInitialMotionX = ev.getX();
                mLastMotionY = mInitialMotionY = ev.getY();
                mActivePointerId = ev.getPointerId(0);
                break;
            }
            case MotionEvent.ACTION_MOVE:
                if (!mIsBeingDragged) {
                    final int pointerIndex = ev.findPointerIndex(mActivePointerId);
                    if (pointerIndex == -1) {
                        // A child has consumed some touch events and put us into an inconsistent
                        // state.
                        needsInvalidate = resetTouch();
                        break;
                    }
                    final float x = ev.getX(pointerIndex);
                    final float xDiff = Math.abs(x - mLastMotionX);
                    final float y = ev.getY(pointerIndex);
                    final float yDiff = Math.abs(y - mLastMotionY);
                    if (DEBUG) {
                        Log.v(TAG, "Moved x to " + x + "," + y + " diff=" + xDiff + "," + yDiff);
                    }
                    if (xDiff > mTouchSlop && xDiff > yDiff) {
                        if (DEBUG) Log.v(TAG, "Starting drag!");
                        mIsBeingDragged = true;
                        requestParentDisallowInterceptTouchEvent(true);
                        mLastMotionX = x - mInitialMotionX > 0 ? mInitialMotionX + mTouchSlop :
                                mInitialMotionX - mTouchSlop;
                        mLastMotionY = y;
                        setScrollState(SCROLL_STATE_DRAGGING);
                        setScrollingCacheEnabled(true);

                        // Disallow Parent Intercept, just in case
                        ViewParent parent = getParent();
                        if (parent != null) {
                            parent.requestDisallowInterceptTouchEvent(true);
                        }
                    }
                }
                // Not else! Note that mIsBeingDragged can be set above.
                if (mIsBeingDragged) {
                    // Scroll to follow the motion event
                    final int activePointerIndex = ev.findPointerIndex(mActivePointerId);
                    final float x = ev.getX(activePointerIndex);
                    needsInvalidate |= performDrag(x);
                }
                break;
            case MotionEvent.ACTION_UP:
                if (mIsBeingDragged) {
                    final VelocityTracker velocityTracker = mVelocityTracker;
                    velocityTracker.computeCurrentVelocity(1000, mMaximumVelocity);
                    int initialVelocity = (int) VelocityTrackerCompat.getXVelocity(
                            velocityTracker, mActivePointerId);
                    mPopulatePending = true;
                    final int width = getClientWidth();
                    final int scrollX = getScrollX();
                    final ItemInfo ii = infoForCurrentScrollPosition();
                    final float marginOffset = (float) mPageMargin / width;
                    final int currentPage = ii.position;
                    final float pageOffset = (((float) scrollX / width) - ii.offset)
                            / (ii.widthFactor + marginOffset);
                    final int activePointerIndex = ev.findPointerIndex(mActivePointerId);
                    final float x = ev.getX(activePointerIndex);
                    final int totalDelta = (int) (x - mInitialMotionX);
                    int nextPage = determineTargetPage(currentPage, pageOffset, initialVelocity,
                            totalDelta);
                    setCurrentItemInternal(nextPage, true, true, initialVelocity);

                    needsInvalidate = resetTouch();
                }
                break;
            case MotionEvent.ACTION_CANCEL:
                if (mIsBeingDragged) {
                    scrollToItem(mCurItem, true, 0, false);
                    needsInvalidate = resetTouch();
                }
                break;
            case MotionEventCompat.ACTION_POINTER_DOWN: {
                final int index = MotionEventCompat.getActionIndex(ev);
                final float x = ev.getX(index);
                mLastMotionX = x;
                mActivePointerId = ev.getPointerId(index);
                break;
            }
            case MotionEventCompat.ACTION_POINTER_UP:
                onSecondaryPointerUp(ev);
                mLastMotionX = ev.getX(ev.findPointerIndex(mActivePointerId));
                break;
        }
        if (needsInvalidate) {
            ViewCompat.postInvalidateOnAnimation(this);
        }
        return true;
    }
```

down事件执行populate 记录当前x,y坐标  move 未拖拽才处理用于滑动view pager 否则由onInterceptTouchEvent处理. 通过滑动速度和x偏移量调用 determineTargetPage 判断下一页位置。

setCurrentItemInternal会去调用populate根据当前位置填充更新mItems并滑动到需要展示页。



---

ViewPager 和PagerAdapter 

```java
public void setAdapter(PagerAdapter adapter) {
        if (mAdapter != null) {
            //老的Adapter中的清理工作 remove  子view
            mAdapter.setViewPagerObserver(null);
            mAdapter.startUpdate(this);
            for (int i = 0; i < mItems.size(); i++) {
                final ItemInfo ii = mItems.get(i);
                mAdapter.destroyItem(this, ii.position, ii.object);
            }
            mAdapter.finishUpdate(this);
            mItems.clear();
            removeNonDecorViews();
            mCurItem = 0;
            scrollTo(0, 0);
        }

        final PagerAdapter oldAdapter = mAdapter;
        mAdapter = adapter;
        mExpectedAdapterCount = 0;

        if (mAdapter != null) {
            if (mObserver == null) {
                mObserver = new PagerObserver();
            }
            //调用mAdapter的notifyDataSetChanged 通过mObserver通知到viewpager
            mAdapter.setViewPagerObserver(mObserver);
            mPopulatePending = false;
            final boolean wasFirstLayout = mFirstLayout;
            mFirstLayout = true;
            mExpectedAdapterCount = mAdapter.getCount();
            if (mRestoredCurItem >= 0) {
                mAdapter.restoreState(mRestoredAdapterState, mRestoredClassLoader);
                setCurrentItemInternal(mRestoredCurItem, false, true);
                mRestoredCurItem = -1;
                mRestoredAdapterState = null;
                mRestoredClassLoader = null;
            } else if (!wasFirstLayout) {
                populate();
            } else {
                requestLayout();
            }
        }

        // Dispatch the change to any listeners
        if (mAdapterChangeListeners != null && !mAdapterChangeListeners.isEmpty()) {
            for (int i = 0, count = mAdapterChangeListeners.size(); i < count; i++) {
                mAdapterChangeListeners.get(i).onAdapterChanged(this, oldAdapter, adapter);
            }
        }
    }
```

PagerAdapter.notifyDataSetChanged 会通知到ViewPager调用dataSetChanged   用于添加删除子view 更新就是先删除再添加

```java
void dataSetChanged() {
        final int adapterCount = mAdapter.getCount();
        mExpectedAdapterCount = adapterCount;
        boolean needPopulate = mItems.size() < mOffscreenPageLimit * 2 + 1
                && mItems.size() < adapterCount;
        int newCurrItem = mCurItem;
        boolean isUpdating = false;
        for (int i = 0; i < mItems.size(); i++) {
            final ItemInfo ii = mItems.get(i);
            //根据mAdapter.getItemPosition返回值来判断子view是否需要重新添加
            //所以可以重写adapter的这个方法来控制子view 是否需要更新重新添加
            final int newPos = mAdapter.getItemPosition(ii.object);
            if (newPos == PagerAdapter.POSITION_UNCHANGED) {
                continue;
            }

            if (newPos == PagerAdapter.POSITION_NONE) {
                mItems.remove(i);
                i--;

                if (!isUpdating) {
                    mAdapter.startUpdate(this);
                    isUpdating = true;
                }

                mAdapter.destroyItem(this, ii.position, ii.object);
                needPopulate = true;

                if (mCurItem == ii.position) {
                    // Keep the current item in the valid range
                    newCurrItem = Math.max(0, Math.min(mCurItem, adapterCount - 1));
                    needPopulate = true;
                }
                continue;
            }
			
            if (ii.position != newPos) {
                if (ii.position == mCurItem) {
                    // Our current item changed position. Follow it.
                    newCurrItem = newPos;
                }

                ii.position = newPos;
                needPopulate = true;
            }
        }

        if (isUpdating) {
            mAdapter.finishUpdate(this);
        }

        Collections.sort(mItems, COMPARATOR);
		//子view顺序有变化需要重新更新
        if (needPopulate) {
            // Reset our known page widths; populate will recompute them.
            final int childCount = getChildCount();
            for (int i = 0; i < childCount; i++) {
                final View child = getChildAt(i);
                final LayoutParams lp = (LayoutParams) child.getLayoutParams();
                if (!lp.isDecor) {
                    lp.widthFactor = 0.f;
                }
            }

            setCurrentItemInternal(newCurrItem, false, true);
            requestLayout();
        }
    }
```

---



ViewPager和TabLayout的联动 

viewpager在滑动及设置显示页面时调用dispatchOnPageScrolled和dispatchOnPageSelected 。tablayout项viewpager set 了TabLayoutOnPageChangeListener用于监听滑动和页面选择。 TabLayout select tab时调用ViewPager的setCurrentItem



---

ViewPager，TabLayout，FragmentStatePagerAdapter使用过程中遇到一个问题
tablayout上跨offset点击 。会导致fragment. getUserVisibleHint状态不对

原因：FragmentStatePagerAdapter destroyItem后会保存fragment信息 其中包括result.putBoolean(FragmentManagerImpl.USER_VISIBLE_HINT_TAG, f.mUserVisibleHint);
此时 f.mUserVisibleHint为false 
下次fragment重建在Fragment.INITIALIZING阶段 
```java
f.mUserVisibleHint = f.mSavedFragmentState.getBoolean(
        FragmentManagerImpl.USER_VISIBLE_HINT_TAG, true);
```
但是此时FragmentStatePagerAdapter的setPrimaryItem  fragment.setUserVisibleHint(true);在这之前执行了所以 所以重建后状态被覆盖为了false。解决方法之一就是让重建的代码先执行再执行 fragment.setUserVisibleHint(true) 可以在这之前加finishUpdate(null)先提交fragment事务
