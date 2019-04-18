### Android Ping工具

需求：

![image-20180917182330654](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20180917182330654.png)产品需要这种功能.

#### 了解android Ping命令 [->](http://aiezu.com/article/linux_ping_command.html)

```shel
ping [-AaDdfnoQqRrv] [-c count] [-G sweepmaxsize]
            [-g sweepminsize] [-h sweepincrsize] [-i wait]
            [-l preload] [-M mask | time] [-m ttl] [-p pattern]
            [-S src_addr] [-s packetsize] [-t timeout][-W waittime]
            [-z tos] host

```

常用的几个参数

| 参数        | 详解                                                     |
| ----------- | -------------------------------------------------------- |
| -c count    | 指定ping的次数                                           |
| -i interval | 设定间隔几秒发送一个ping包，默认一秒ping一次 单位秒      |
| -f          | 极限检测。大量且快速地送网络封包给一台机器，看它的回应。 |
| -q          | 不显示任何传送封包的信息，只显示最后的结果               |
| -w          | ping持续时间 单位秒                                      |

1. 忽略ping时间对应 -i 0.2 最小0.2
2. 不分隔数据包对应-f 
3. 永远ping 对应是否有时间-w  或次数-c限制（我觉的最好限制一下,目前是通过正则查整个数据获取ping统计）
4. ping时间间隔对应-i 
5. ping次数对应-c

#### Android执行Shell命令

```java
package cn.mwee.android.skynet.ui.ping;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;

import io.reactivex.Observable;
import io.reactivex.Observer;
import io.reactivex.disposables.Disposable;
import io.reactivex.subjects.PublishSubject;

/**
 * Description:
 *
 * @author zhou.junyou
 * Create by:Android Studio
 * Date:2018/9/17
 */
public class ExcutePingObservable extends Observable<String> implements Disposable {
    private static final String LINE_SEP = System.getProperty("line.separator");
    public PublishSubject<String> logSubject;
    public String command;
    protected Process logcatProcess;
    protected BufferedReader reader;
    protected BufferedReader errorResult;

    public ExcutePingObservable(PublishSubject<String> logSubject, String command) {
        this.logSubject = logSubject;
        this.command = command;
    }

    @Override
    protected void subscribeActual(Observer observer) {
        observer.onSubscribe(this);
        observer.onNext(excuteCommand(command));
        observer.onComplete();
    }

    @Override
    public void dispose() {
       closeResource();
    }

    @Override
    public boolean isDisposed() {
        return logcatProcess==null;
    }

    private String excuteCommand(String command) {
        StringBuilder result = new StringBuilder();
        logcatProcess = null;
        reader = null;

        try {
            logcatProcess = Runtime.getRuntime().exec(command);

            reader = new BufferedReader(new InputStreamReader(logcatProcess.getInputStream()));
            errorResult = new BufferedReader(new InputStreamReader(logcatProcess.getErrorStream(), "UTF-8")
            );
            String line;
            logSubject.onNext("执行命令: "+command);
            while ((line = reader.readLine()) != null&&logcatProcess!=null) {
                result.append(LINE_SEP).append(line);
                logSubject.onNext(line);
            }
            while ((line = errorResult.readLine()) != null&&logcatProcess!=null) {
                result.append(LINE_SEP).append(line);
                logSubject.onNext(line);
            }
        } catch (SecurityException |
                IllegalArgumentException |
                NullPointerException |
                IOException e) {
            e.printStackTrace();
        } finally {
            closeResource();
        }
        return result.toString();
    }

    private void closeResource() {
        if (logcatProcess != null) {
            logcatProcess.destroy();
        }
        logcatProcess=null;
        if (reader != null) {
            try {
                reader.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        if (errorResult != null) {
            try {
                errorResult.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }


    }
}

```

简单的想法

1. logSubject  用于上面展示命令行日志 每读取一行就显示到界面上

2. 自己实现Observable 主要为了实现停止任务的时候比较方便 只需在外面rx流最后调用下dispose.dispose()

   还有和外面fragment生命周期也绑定了一下 退出界面也停止任务

3. 将最后的result

#### Ping返回

![image-20180917195857550](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20180917195857550.png)

29个包被发送 28个已接受 丢包率3% 耗时5628 最小延迟74.731ms 最大延迟123.988毫秒 平均延迟92.751毫秒

mdev是 Mean Deviation 这个值越大说明网络越不稳定

这部分依靠正则匹配解析到对象

```java
   static Pattern p1 = Pattern.compile("[0-9]*\\.[0-9]*/[0-9]*\\.[0-9]*/[0-9]*\\.[0-9]*/[0-9]*\\.[0-9]* [a-z]*");
  static   Pattern p2 = Pattern.compile("[0-9]*% packet loss");
public static LatencyResult findLatency(String result) {
        LatencyResult latencyResult = new LatencyResult();
        Matcher matcher = p1.matcher(result);
        if (matcher.find()) {
            String group = matcher.group();
            String[] pingGroup = group.split("/");
            if (pingGroup.length == 0) {
                return latencyResult;
            }
            latencyResult.setMinPing(parseFloat(pingGroup[0]));
            latencyResult.setAvgPing(parseFloat(pingGroup[1]));
            latencyResult.setMaxPing(parseFloat(pingGroup[2]));
            /**
             * 我么的搞了，我搞不定了
             */
            if (pingGroup[3] != null && pingGroup[3].endsWith("ms")) {
                pingGroup[3] = pingGroup[3].substring(0, pingGroup[3].length() - 3);
            }
            latencyResult.setMdev((parseFloat(pingGroup[3])));
        }
        Matcher lossMatcher = p2.matcher(result);
        if (lossMatcher.find()) {
            String group = lossMatcher.group(0);
            String substring = group.substring(0, group.indexOf("%"));
            latencyResult.setLossPercent(Float.parseFloat(substring));
        }
        return latencyResult;
    
    
     private static float parseFloat(String ping) {
        try {
            if (TextUtils.isEmpty(ping)) {
                return 0f;
            }

            return Float.parseFloat(ping);
        } catch (Exception v0) {
            v0.printStackTrace();
        }

        return 0f;
    }

    ///////////////////////////////////////////////////////////////////////////
    //
    ///////////////////////////////////////////////////////////////////////////
    public static class LatencyResult implements Serializable {
        private float minPing;
        private float avgPing;
        private float maxPing;
        private float mdev;
        private float lossPercent;

        public float getMinPing() {
            return minPing;
        }

        public void setMinPing(float minPing) {
            this.minPing = minPing;
        }

        public float getAvgPing() {
            return avgPing;
        }

        public void setAvgPing(float avgPing) {
            this.avgPing = avgPing;
        }

        public float getMaxPing() {
            return maxPing;
        }

        public void setMaxPing(float maxPing) {
            this.maxPing = maxPing;
        }

        public float getMdev() {
            return mdev;
        }

        public void setMdev(float mdev) {
            this.mdev = mdev;
        }

        public float getLossPercent() {
            return lossPercent;
        }

        public void setLossPercent(float lossPercent) {
            this.lossPercent = lossPercent;
        }
    }
```

####界面的一些注意点

1. 界面上日志列表数据不能存过多 可以限制成超过500条删除100条
2. 有新日志产生时滑倒最后 
3. 用户查看日志的时候不要自动滑动 

```java
 rvLog.addOnScrollListener(new RecyclerView.OnScrollListener() {
            @Override
            public void onScrollStateChanged(RecyclerView recyclerView, int newState) {
                super.onScrollStateChanged(recyclerView, newState);
            }

            @Override
            public void onScrolled(RecyclerView recyclerView, int dx, int dy) {
                super.onScrolled(recyclerView, dx, dy);
                // update what the first viewable item is
                final LinearLayoutManager layoutManager = (LinearLayoutManager) recyclerView.getLayoutManager();

                // if the bottom of the list isn't visible anymore, then stop autoscrolling
                mAutoscrollToBottom = (layoutManager.findLastCompletelyVisibleItemPosition() == recyclerView.getAdapter().getItemCount() - 1);
            }
        });
Disposable subscribe = logSubject.observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Consumer<String>() {
                    @Override
                    public void accept(String s) throws Exception {
                        logAdapter.addData(s);
                        int max = 500;
                        if (logAdapter.getData().size() > max) {
                            logAdapter.removeFirst(100);
                        }
                        if (mAutoscrollToBottom) {
                            rvLog.scrollToPosition(logAdapter.getItemCount() - 1);
                        }
                    }
                });
```



#### 建议及问题

1. ping间隔改成秒默认1秒  android shell  ping -i 是秒  ,ping次数建议默认4-10次， 永远ping 建议不要永远几百次够了,经过时间建议改成秒，忽略ping最小间隔200ms

2. 目前统计分析是ping完成后 解析ping结果的并不是每返回一次ping结果计算一次。

3. 关于交互

   1. 界面上日志列表数据不能存过多 可以限制成超过500条删除100条
   2. 有新日志产生时滑倒最后 
   3. 用户查看日志的时候不要自动滑动 












