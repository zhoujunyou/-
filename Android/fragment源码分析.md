### fragment源码分析

基于android.support.v4.app 27.1.1

**fragment相对于View来说 最主要的就是多了生命周期函数 先对应FragmentActivity 分析何时执行函数**

![image-20190104173142544](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20190104173142544.png)简单的一个类图

* FragmentController.是FragmentActivity成员变量 用于控制FragmentManager其持有HostCallBacks 
* HostCallBacks。持有FragmentManager及Activity。
* FragmentManager。及实现类为FragmentManagerImpl用于管理fragments 添加删除更改状态等操作
* FragmentTransaction其实现类为BackStackRecord 用于保存fragment的事务及回退栈 commit后由FragmentManagerImpl进行具体操作
*  FragmentContainer: FragmentHostCallback的父类用于寻找容器。

| FragmentActivity | FragmentController | FragmentManager     | Fragment                                                     |
| ---------------- | ------------------ | ------------------- | ------------------------------------------------------------ |
| onCreate         | attachHost         | dispatchStateChange | onAttach,onCreate                                            |
| onStart          | dispatchStart      | dispatchStateChange | onCreateView,onViewCreated,onActivityCreated,restoreViewState |
| onResume         | dispatchResume     | dispatchStateChange | onStart,onResume                                             |
| onPause          | dispatchPause      | dispatchStateChange | onPause                                                      |
| onStop           | dispatchStop       | dispatchStateChange | onStop,onDestroyView                                         |
| onDestroy        | dispatchDestroy    | dispatchStateChange | onDestroy,onDetach                                           |

| perform                | state                                          |
| ---------------------- | ---------------------------------------------- |
| performCreate          | mState = CREATED;mIsCreated = true;            |
| performCreateView      | mPerformedCreateView = true;                   |
| performActivityCreated | mState = ACTIVITY_CREATED;                     |
| performStart           | mState = STARTED;                              |
| performResume          | mState = RESUMED;                              |
| performPause           | mState = STARTED;                              |
| performStop            | mState = STOPPED;                              |
| performReallyStop      | mState = ACTIVITY_CREATED;                     |
| performDestroyView     | mState = CREATED;mPerformedCreateView = false; |
| performDestroy         | mState = INITIALIZING;mIsCreated = false;      |
| performDetach          |                                                |



#### Fragment Commit 操作

1. FragmentManager.beginTransaction 返回BackStackRecord

2. add,replace等操作会先添加到ArrayList<Op> mOps中

3. 一个BackStackRecord只能commit一次，将OpGenerator加入mPendingActions。mainHandler提交mExecCommit任务，mTmpRecords中加入当前BackStackRecord，expandOps(ArrayList<Fragment> added, Fragment oldPrimaryNav)对added及mOps整理， executeOps()操作FragmentManager及其中的fragment,最后通过moveFragmentToExpectedState(Fragment f)进行fragment状态切换。状态大致按照FragmentManager的mCurState，一些特殊情况按照Fragment当前状态做一个转换比如f.mRemoving，f.mDeferStart。show和hide通过是否f.mHiddenChanged 在moveFragmentToExpectedState最后执行

    completeShowHideFragment就是对fragment.mView进行Visible,Gone操作,add和remove通过f.mContainer 的addView和removeView进行界面操作。



####  FragmentTransaction commit,commitAllowingStateLoss,commitNow,commitNowAllowingStateLoss区别
1. commit 将OpGenerator加入mPendingActions  先进行checkStateLoss检测。官方描述(**您只能在 Activity 保存其状态（用户离开 Activity）之前使用 commit() 提交事务。如果您试图在该时间点后提交，则会引发异常。 这是因为如需恢复 Activity，则提交后的状态可能会丢失。 对于丢失提交无关紧要的情况，请使用 commitAllowingStateLoss()**)  execPendingActions任务提交到handler 也就是说 execPendingActions不会立即执行  等handler次数队列中的任务先执行完毕。
2. commitAllowingStateLoss  相对于commit 少了checkStateLoss这一步。
3. commitNow 不允许事物有addToBackStack操作 相对于commit 不是通过handler 而是调用execSingleAction立即执行
4. commitNowAllowingStateLoss  相对于commitNow 少了checkStateLoss

 **FragmentManager executePendingTransactions** ：立即调用 execPendingActions，mPostponedTransactions中的action



**popBackStack,popBackStackImmediate,popBackStack(String name, int flags),popBackStackImmediate(String name, int flags)的区别**

1. popBackStack  与commit类似异步提交popBackStackState 弹出mBackStack最后一个record 执行

    record.executePopOps(moveToState); 

2. popBackStack(String name, int flags) .弹出与addToBackStack(@Nullable String name) name相同mBackStack倒数第一个record之后的record flags=POP_BACK_STACK_INCLUSIVE 继续向前一个寻找是否有连续相同name 将其之后的record一起弹出

3.  popBackStackImmediate 立即执行 弹出最后一个record 需要checkStateLoss

4. popBackStackImmediate(String name, int flags)  与popBackStack(String name, int flags)相对应只是立即去执行的

如果setPrimaryNavigationFragment 设置了该fragment pop操作先会从mChildFragmentManager开始 



#### Fragment.setTargetFragment

用于两个fragment调用onActivityResulttong x



#### FragmentManager  detach和remove的区别 

remove方法会走到 fragment的onDestroy和onDetach 调用到makeInactive 将mActive中该fragment清除

detach后fragment走到onDestroyView 不会清除mActivte中的fragment,可以重新找到并attac



#### Fragment重叠问题

* 这个问题一般发生在Activity回复重建时。 在FragmentActivity中onSaveInstanceState会调用FragmentManager的saveAllState  会讲mAtive中所有Fragment的状态存到FragmentManagerState再保存到outState中 键为android:support:fragments

  在FragmentActivity的onCreate中 会调用FragmentManager的restoreAllState 将FragmentManagerState从outState取出。重新赋值到FragmentManager中 重走生命周期。

  在api 24.0.0之前FragmentState中没有保存mHidden值 所以重建的fragment这个值都是为false 而我们View的显示隐藏都用这个值判断 所以用show hide重建时导致重叠。还有个原因就是如果addfragment时没有判断

  FragmentManager中是否已经有相同类型的Fragment 就会导致存在两个Fragment  所以需要在Activity的onCreate方法中判断一下outstate是否为null



---

记一次Fragment重建导致的异常。
```java

                            if (mActive.get(f.mTarget.mIndex) != f.mTarget) {
                                throw new IllegalStateException("Fragment " + f
                                        + " declared target fragment " + f.mTarget
                                        + " that does not belong to this FragmentManager!");
                            }
                         

```
美小易PrinterConfigContainerActivity 中在onCreate中 用replace commit了

```java
mTPrinterSetFragment = new TPrinterSetFragment();
FragmentTransaction transaction = getSupportFragmentManager().beginTransaction();
transaction.replace(R.id.mAppConfigContainer, mTPrinterSetFragment);
transaction.commit();
```

 TPrinterSetFragment的onViewCreated中调用了

 final Progress progress = ProgressManager.showProgress(this);

activty刚创建时一切正常。但是activty重启(可以在开发者模式中选择不保留活动模拟)   时。就会抛出

**Caused by: java.lang.IllegalStateException: Fragment Progress{686714 #3 com.mwee.android.pos.air.business.tprinter.TPrinterSetFragment_progress} declared target fragment TPrinterSetFragment{9542bbd} that does not belong to this FragmentManager!**

* 分析过程
   Activity被销毁前调用了saveInstanceState 保留了fragment1状态   在重启时 onCreate方法中执行Fragmentmanager的restoreAllState 将原有的fragment1恢复 ,调用了commit 将任务提交到handler  ,。Activity走到onStart方法 调用mFragments.dispatchStart(); 。fragment1的onViewCreated被调用，走到ProgressManager.showProgress(this) 提交一个了一个progress1的Fragment。接着 replace事务被执行,先执行fragment1的remove。到后面progress的事务被执行 
    此时progress1的target fragment1已被移除， 所以mActive.get(f.mTarget.mIndex)==null了。 

* 解决方法

   其实这个问题 本质上还是因为没考虑到actvity 重启。在oncreate中加个判断 if (savedInstanceState == null) 才去重新创建fragment. 否则从fragmentManager中根据id查出来。








