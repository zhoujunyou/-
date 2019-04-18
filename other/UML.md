### UML类图

---

为啥：做笔记想写个类图 TMD的居然不会 这里先弥补一下[先mark一下](https://www.kancloud.cn/digest/xing-designpattern/143734)

####类组成

1. 一般类

![盗用](https://box.kancloud.cn/2016-04-21_571890ed22d4c.jpg)

 1. 上中下部分分别代表类名 ，属性 ，方法。

 2. \+,\#,\- 分别代表可见性public, protect,private ,后面跟的是属性名或方法名

 3. 属性的格式：可见性  名称:类型 [ = 缺省值 ]

 4. 方法的格式： 可见性  名称(参数列表) [ : 返回类型]

 5. 下划线表示静态方法 



2. 抽象类

![陶都](https://box.kancloud.cn/2016-04-21_571890ed356a1.jpg)

区别就是方法和类名用了斜体

3. 接口

![这里写图片描述](https://box.kancloud.cn/2016-04-21_571890ed4697e.jpg)

多了个interface





####类关系连接线    

![image-20180915104940364](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20180915104940364.png)聚合

![image-20180915105011816](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20180915105011816.png)组合

![image-20180915105032699](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20180915105032699.png)继承

![image-20180915105147322](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20180915105147322.png)关联

![image-20180915105215179](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20180915105215179.png)依赖

![image-20180915105503267](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20180915105503267.png)实现

**泛化= 实现> 组合> 聚合> 关联> 依赖**





​     

