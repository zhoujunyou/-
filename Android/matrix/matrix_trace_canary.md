### matrix_trace_canary

问题方法检测原理：在class文件转dex文件时 扫描class在需要修改的地方利用asm修改字节码文件

#####打包脚本逻辑

1. 利用matrix_gradle_plugin 将transformClassesWithDexBuilder或者transformClassesWithDex task中的TransformTask的transform利用反射替换成MatrixTraceTransform。这样先会走到MatrixTraceTransform的transform方法中。

2. 先收集源码目录及jar包中收集我们需要的文件。可以通过blackMethodList.txt文件过滤 默认过滤规则

   ```
    "[package]\n"
                       + "-keeppackage android/\n"
                       + "-keeppackage com/tencent/matrix/\n";
   ```

   我们可以用-keepclass 和-keeppackage规则控制不需要修改的文件。

3. 从收集的文件中收集方法，jar包中过滤非class文件 及包含"R.class", "R$", "Manifest", "BuildConfig"类型文件.通过org.objectweb.asm.ClassVisitor的visit方法收集.这里逻辑涉及到MethodCollector下的SingleTraceClassAdapter，TraceClassAdapter，CollectMethodNode，

4. 将需要修改的方法记录到build/matrix_output/Debug.methodmap下。拿到异常方法id后可到这里去查

   将忽略的方法记录到build/matrix_output/Debug.ignoremethodmap下

5. 调用MethodTracer的trace方法去修改class文件中的方法，MethodTracer下的TraceClassAdapter，逻辑。如果是activiy的子类复写onWindowFocusChanged方法 。TraceMethodAdapter 进入方法时调用com/tencent/matrix/trace/core/MethodBeat的i方法。方法结束前判断是不是onWindowFocusChanged方法。如果是调用MethodBeat的at方法 ，否则调用MethodBeat.o方法

6. 最后使用原来的Transform去做class转dex的逻辑 转换后的class保留在build/matrix_output/classes下

   后续的逻辑就在MethodBeat下了。



####具体各种trace.

* **FrameTracer**: FrameBeat中使用Choreographer.postFrameCallback 记录doFrame时间 如果正在onDraw且与上次doFrame时间做比较间隔时间超过16.66ms调用丢帧回调 记录。
* **FPSTracer：**同样用FrameBeat监听 doFrame方法中先判断当前activity是resume且onDraw。记录doFrame间隔时间及当前activity及fragment  加入mFrameDataList 。 mLazyScheduler论询检测mFrameDataList是否为空 不为空则计算当前的FPS及丢帧等级。
