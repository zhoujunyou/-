#### 全能pos jenkins打包

1. http://jks.mwbyd.cn/jenkins/
2. 用户名：zhou.junyou  密码：zhou.junyou
3. SmartPay->Build with Parameters ->  mType:all  ,pay_branch:,cateringbranch:->开始构建
4. apk路径： http://svn.mwbyd.cn/APK/SmartPay/
5. SVN账号：zhou.junyou 密码：zjy@1qaz

#### 全能pos 自更新配置

1. http://mnr.mwbyd.cn/index.php/index/Remotecontrol/edit?t=1536573214  进入中控
2. 点击首页->添加软件->版本类型选择全能pos或全能pos-拉卡拉版->选择发布渠道->填写更新描述->上传apk
3.  apk类型 拉卡拉版选择SmartPay-mwee-releaseLakala   非拉卡拉版选择SmartPay-smartque-releaseNoBoot
4. 提交->等待跳转软件列表或手动进入软件列表->查看选择策略->通过审核

#### 全能POS发布

1. 准备发布包
   1.  [拉卡拉平台](http://open.lakala.com/)：SmartPay-mwee-releaseLakala ,SmartPay-smartque-releaseNoBoot
   2.  [慧银平台](https://www.wizarview.cn/wizarView/index.action)：SmartPay-smartque-release
   3.  [美味中控](http://mnr.mwbyd.cn/index.php/index/Remotecontrol/edit?t=1536579258)：拉卡拉版选择SmartPay-mwee-releaseLakala   非拉卡拉版选择SmartPay-smartque-releaseNoBoot
   4.  [商米](<http://partner.sunmi.com/>)  ： SmartPay-mwee-release

####拉卡拉P960

收单版本号后缀不是.0, 不小心升级后得恢复出厂设置 。(最好能获取拉卡拉版本号 与设备不匹配不让用银联收款)

#### 第三方美小二分支
pay2.9.4_v3.4.4 | Release_SmartPay_V2.9.4





