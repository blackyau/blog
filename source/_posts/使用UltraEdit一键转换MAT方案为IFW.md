---
layout: post
title: 使用UltraEdit一键转换MAT方案为IFW
date: 2017-09-17 23:09:00
updated: 2017-09-18 15:26:00
categories: 教程
tags: 
    - MAT
    - IFW
    - My Android Tools
    - Intent Firewall
    - UltraEdit
    - 宏
    - 一键
urlname: 8
comment: true
---
使用 Intent Firewall 可以让你优雅的做到类似 My Android Tools 禁用组件的效果

<!-- more -->

My Android Tools作为一位Android优化软件新秀，使用与其他优化软件单纯的杀后台抑制后台唤醒完全不同的方法，深受Android倒腾党的喜爱。但任何应用App都可以使用公开的接口[Android Developer](https://developer.android.com/reference/android/content/pm/PackageManager.html?hl=zh-cn#setComponentEnabledSetting)来重新激活自己的组件。My Android Tools作者也考虑到了这个问题，同时也推出了一个[Xposed模块](https://www.coolapk.com/apk/cn.wq.myandroidtoolsxposed)阻止这个api的调用。

Xposed模块的滥用又会导致系统执行效率，详细原因可以查看该文[为什么安装 Xposed 以后会导致卡顿](https://blog.nfz.moe/archives/why-xposed-cause-unsmooth-exprience.html)。所以又有人挖掘除了新的方法，能够在仅ROOT的情况下阻止禁用后的Service/Receiver/Activity调用api重新激活自己的组件。绿色守护在V3.0时也在新推出的“处方”功能时，使用了该特性。**Intent Firewall**

## Intent是什么？
Activity、服务和广播接收器，是通过名为 Intent 的消息进行启动的。可以将 Intent 视为从其他组件请求操作的信使，无论组件属于这个应用还是其他应用。
对于 Activity 和服务， Intent 传达要执行的操作。例如， Intent 传达的请求可以是打开应用设置界面的Activity，或是启动下载某文件的服务
对于广播接收器， Intent 只会定义要广播的通知，可以响应这个广播的接收器会被启动以执行下一步操作

## Intent Firewall的原理？
写轮眼是通过关闭组件来达到效果，而处方则是拦截了启动组件的途径。Intent防火墙是Android框架的一个组件，允许根据XML文件中定义的规则强制阻止intent。Intent防火墙不是Android框架的官方支持的功能，没有官方文档，只能通过阅读源代码了解。
每一个在Android框架中启动的Intent，包括由操作系统创建的Intent，都会通过Intent防火墙。这意味着Intent防火墙有权允许或拒绝任何Intent。Intent防火墙在决定如何处理传入Intent时不考虑发送方，只考虑Intent的细节和其预期的接收者。XML文件被写入到`/data/system/ifw`目录并可以删除，这使得Intent防火墙可以动态更新其规则集。
绿色守护在此基础上进行开发，使处方成为一个非常强大的功能，但严重依赖网友贡献的意图筛选规则，比My Android Tools更甚。

## Intent Firewall的编写规则？
你可以参考[http://www.cis.syr.edu/~wedu/android/IntentFirewall/](http://www.cis.syr.edu/~wedu/android/IntentFirewall/)。下面是一个实例

```xml
<rules>
 <broadcast block="true" log="false">
  <component-filter name="com.tencent.mobileqq/cooperation.dingdong.DingdongPluginProxyBroadcastReceiver" />
  <component-filter name="com.tencent.mobileqq/cooperation.qzone.QzoneProxyReceiver" />
  <component-filter name="com.tencent.mobileqq/com.tencent.open.downloadnew.common.DownloadReceiver" />
  <component-filter name="com.tencent.mobileqq/com.tencent.open.downloadnew.common.DownloadReceiverWebProcess" />
  <component-filter name="com.tencent.mobileqq/com.tencent.open.business.base.appreport.AppReportReceiver" />
  <component-filter name="com.tencent.mobileqq/cooperation.weiyun.WeiyunProxyBroadcastReceiver" />
  <component-filter name="com.tencent.mobileqq/cooperation.weiyun.WeiyunBroadcastReceiver" />
  <component-filter name="com.tencent.mobileqq/com.tencent.mobileqq.msf.core.NetConnInfoCenter" />
</broadcast>
 <service block="true" log="false">
  <component-filter name="com.tencent.mobileqq/com.tencent.mobileqq.app.CoreService" />
  <component-filter name="com.tencent.mobileqq/com.tencent.mobileqq.app.CoreService$KernelService" />
  <component-filter name="com.tencent.mobileqq/cooperation.qzone.remote.logic.QzoneWebPluginProxyService" />
  <component-filter name="com.tencent.mobileqq/com.tencent.tmdownloader.TMAssistantDownloadService" />
</service>
 <activity block="true" log="false">
  <component-filter name="com.tencent.mobileqq/com.tencent.mobileqq.activity.UpgradeActivity" />
  <component-filter name="com.tencent.mobileqq/com.tencent.mobileqq.activity.UpgradeDetailActivity" />
 </activity>
</rules>
```

`broadcast`为广播，`service`为服务，`activity`为活动。其他的你应该很容易的看出规律。

## 如何将MAT备份转为IFW方案？
首先下载安装UltraEdit个人推荐zd426破解版，因主站易主不建议使用最新版，这里提供可信的历史版直链下载(基于又拍云的个人CDN)和蓝奏云
[UltraEdit_v25.0.0.82_x64_zh_CN](https://st.blackyau.net/dl/UltraEdit/UltraEdit_v25.0.0.82_x64_zh_CN.7z)
[UltraEdit_v25.0.0.82_x32_zh_CN](https://st.blackyau.net/dl/UltraEdit/UltraEdit_v25.0.0.82_x32_zh_CN.7z)
[UltraEditv 8.20 简体中文汉化经典版单文件](https://st.blackyau.net/dl/UltraEdit/UltraEdit.exe)
[https://www.lanzous.com/b275916](https://www.lanzous.com/b275916) 密码:hg3y

然后下载[MAT转IFW.mac](https://dl.blackyau.cc/?dir=%E7%A6%81%E7%94%A8%E6%96%B9%E6%A1%88)
![1](https://st.blackyau.net/blog/8/1.png)

**在转换时不建议使用键盘或鼠标进行任何操作，在转换量较大时应用会未响应属正常现象。等待即可。**
目前在`MAT转IFW-new`中使用了新的思路，来自[cubesky](https://bbs.letitfly.me/d/100/11)，转换速度快到令人发指。实现方法为，先在两边加上前缀和后缀，后将整个MAT方案复制两遍然后在将IFW用于判定broadcast、service、activity属性的语句，重复三遍。

```xml
<rules>
 <broadcast block="true" log="false">
    我是毒瘤服务1
    我是毒瘤服务2
    我是毒瘤广播1
    我是毒瘤广播2
    我是毒瘤活动1
</broadcast>

 <service block="true" log="false">
    我是毒瘤服务1
    我是毒瘤服务2
    我是毒瘤广播1
    我是毒瘤广播2
    我是毒瘤活动1
</service>

 <activity block="true" log="false">
    我是毒瘤服务1
    我是毒瘤服务2
    我是毒瘤广播1
    我是毒瘤广播2
    我是毒瘤活动1
 </activity>
</rules>
```

经测试，方案中的所有服务/广播/活动都被IFW正常干掉。具体会导致什么负面效果有待进一步的查证，现在看来除了会让IFW配置文件变得大一点还没发现什么问题。

`MAT转IFW-old`使用了broadcast、service、activity关键字来判断组件属性，生成IFW方案比较干净没有多余的内容。不过如果有不按标准命名的组件，将会在宏运行结束后在最底部需要你手动操作一下。你可以将光标停留在任意一行，通过快捷键ALT+F1|ALT+F2|ALT+F3分别将该行组件插入服务|广播|活动分组。

我正在寻找能够高效率/跨平台完成MAT到IFW的转换方法，如果你有建议欢迎给我提出。这会对我有极大的帮助。

如果您由于各种原因，无法自己转换。可以将MAT方案发送到 blackyau426@gmail.com 、Telegram:@Black_Yau或是直接在博客评论区留言。我会尽力为你解答。