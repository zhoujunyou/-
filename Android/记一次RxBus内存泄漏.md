### 记一次RxBus内存泄漏

问题: bugly出现大量ANR  分析可能是大量的listfiles操作引起的。

![image-20181218104212055](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20181218104212055.png)

![image-20181218104226832](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20181218104226832.png)



#### 解决步骤

1. 根据账号设备号找到对应的设备日志。![image-20181218104510552](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20181218104510552.png)
2. 分析日志发现在第三方调用支付时有大量的rxbus日志New event received。怀疑这个对象一直在创建未被释放。
   ![image-20181218104602155](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20181218104602155.png)
3. 模拟第三方支付，果然发现日志越打越多 每次调用都比 上一次多打，dump内存查看这个对象个数发现越来越多![image-20181218105216841](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20181218105216841.png)
4. 分析引用发现SmartMain$1一直在创建subscribe Rx2Bus\$1![image-20181218105831410](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20181218105831410.png)
5. 找到了罪魁祸首 SmartMain是应用主页一直未调用onDestroy 而每次第三方调用都会到onNewIntent 也就是每调用一次 泄漏一个Rx2Bus$1。![image-20181218110040069](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20181218110040069.png)
6. 修复后重新执行第三方支付 对象数量正常。

