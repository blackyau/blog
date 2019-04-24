---
layout: post
title: MicroG安装与配置
date: 2016-09-30 16:27:00
updated: 2018-03-25 15:26:00
categories: 教程
tags: 
    - Android
    - MicroG
    - Google
    - 安装
    - 配置
urlname: 4
comment: true
---
轻量级 Google 全家桶，MicroG 安装与配置教程

<!-- more -->

Google全家桶一直是让Android玩家们又爱又恨的组件，一方面它能够带给我们完整的Google生态圈体验、一方面又被他频繁唤醒设备造成的高耗电和全家桶的臃肿所烦恼。而现在有另外一个“不完美”的解决方案，可以让我们用上Google play、让Google全家桶能够安静的不再频繁唤醒设备。那就是**[MicroG项目](https://microg.org)**，又得益于这个项目为开源所以使用MicroG也可以让我们摆脱Google对Android“可远观而不可亵玩”那种开放。

安装须知
 - 需要完整的ROOT权限
 - 需要安装xposed框架58版本以上
 - 需要Android核心破解
 - 需要一颗爱折腾的心

安装步骤

**1.下载欺骗所用的xposed模块**
 - 打开Xposed Installer当中的“下载”搜索DisableFlagSecure（欺骗Google并使用MicroG接管GAPPS相关工作）和FakeGApps（使用MicroG定位所需模块）。
 - 下载并安装以上两个模块，打钩后重启。

**2.配置f-droid**(用于接收MicroG组件更新)
 - 在[f-droid.org](https://f-droid.org)当中下载并安装应用（蓝色的Download F-Droid）
 - 在应用主页中点击 右上角的垂直三点 -管理软件源-右上角+号
 - 在添加新的软件源窗口中，软件源地址填写“microg.org/fdroid/repo”指纹请留空
 - 点击添加后再点击 右上角的顺时针箭头 更新资源库
 - 回到应用主页面将“最新”调整为“全部”，点击右上角的 放大镜 输入“nlp”，下载并安装以下所有应用
    GSMLocationNlpBackend
    LocalGsmNlpBackend
    LocalWifiNlpBackend
    MozillaNlpBackend
    OpenBmapNlpBackend

![img](https://st.blackyau.net/blog/4/1.png)

**3.配置MicroG**
 - [进入MicroG组件下载页](https://microg.org/download.html)下载并安装microG Services Framework Proxy过后在下载安装microG Services Core（请注意先后顺序，因为先安装core部分设备可能会找不到组件）两个都安装完毕后请重启。
 - 打开应用microG Setting将图中两个选项全部打钩![img](https://st.blackyau.net/blog/4/2.jpg)
 - 滑动到下方点击UnifiedNlp Settings检查一下Configure location backends选项中是否有，刚刚在配置f-droid时下载的应用。如果有的话请全部打钩并点击确定。
 - 回到microG Setting应用主页，点击第一个Self-Check。此时应该看到

第一个"Signature Spoofing Support(签名欺骗支持)"群组的
System Grants Signature Spoofing permission<-(Android 6.0+)已经打勾
System has Signature Spoofing support<-(Android 6.0+)已经打勾
System Spoofs Signature<-已经打勾

第四个群组"Network Location Provider Support(网路本地提供者支持)"则会看到
Android Version Supported<-已经打勾(低于4.0则无勾)
System supports location provider<-已经打勾(没定位功能的装置则无勾)

第五个群组"UnifiedNlp Status(模组状态)"则会看到UnifiedNlp is Registered in System<-没勾，需要全部设定好后重新开机才会打勾(有时就算全部设定好重开不见得会打勾= 3=)
Location backend(s) set up<-已经打勾(若在配置MicroG第3步时没有发现组件或是组件至少没有打勾一个，此处则无勾)
Network-Based location enabled(定位高精确度已启用)<-没勾

<center>-----若设备没有定位功能可省略-----</center>

 - 启动设备的“定位服务”并选择到高精度（请勿贪图省电而选择省电，APP会不鸟你）
 - 启动Xposed Installer到模块中打开XposedGmsCoreUuifiedNlp
 - 点选唯一的选项"Check Setting(检查设定)"
 - 若成功，会显示两行绿色(第三行若出现红色的错误提示可以忽略，因为这个提示的表示找不到你的"位置")
 - 若在第一行就出现失败，代表你可能多安装并启用了XposedUnifiedNlp，这个是程序对于已安装了官方的GAPPS才需要用到的，请取消

<center>-----若设备没有定位功能可省略-----</center>

 - 下载google play商店 可以到apkmirror中下载[点我可转跳到下载页](https://www.apkmirror.com/apk/google-inc/google-play-store/)
 - 使用任何一个具有ROOT权限的文件管理器，将play商城的APK移动到/system/app目录当中，并将权限设置为如图所示![img](https://st.blackyau.net/blog/4/3.png)
 - 点一下该Apk文件确认可以跳出安装画面(请不要再安装一次！！！)。若可以，请重新启动。即可正常使用play商店

**4.安装地图API支持**
 - 到[此处](https://github.com/microg/android_frameworks_mapsv1/releases)下载zip包，并使用第三方Recovery刷入
 - 若刷机失败，请解压缩刷机包，将"System"文件夹夹复制到根目录(/)，没意外的话档桉管理器会提示出即将覆盖资料夹(请选"是"将其覆盖)，然后依照刷机包内的结构目录，把文件夹的权限改为三读一写(644)。

**已知的问题：**
 - 通讯录和日历仍然需要GAPPS的支持
 - 部分应用程式若有同步功能仍然会提示"Google Play 服务 版本过旧，请更新到最新版本"(之类的字词)，造成软件无法进行同步(Google 云端硬盘 Gmail等等...)


**通讯录和日历同步解决方案：**
 - 在 [opengapps.org](https://opengapps.org/) 中下载pico版
 - 从压缩包根目录的"GApps"资料夹解压缩"calsync-all.tar.xz"，然后再解压缩"calsync-all.tar.xz"，把里面唯一的APK解压出来(此文件为 Google 日历同步的APK)
 - 从压缩包根目录的"Core"资料夹把"googlecontactssync-all.tar.xz"和"googlebackuptransport-all.tar.xz"解压缩出来(前者为 Google 通讯录同步 ，后者为 Google备份传输[Google Backup Transport]，每个压缩档各只有一个文件)
 - 把 Google 通讯录同步 和 Google 日历同步放到/system/app/
 - 再把 Google备份传输 放到 /system/priv-app
 - 重新开机，然后尝试同步看看吧。因为XDA论坛上面有人成功了。
PS:下面路径都是从下载回来的压缩包路径开始讲解。注意，从这个地方下载的文件都是多重压缩包桉所制作而成的[也就是压缩包里面还有压缩包]
PSS:TAR.XZ 为 TAR ，TAR为一般的压缩档。如果你使用的文件管理器无法识别此格式，请手动指定文件管理器用压缩包格式开启或是用[其他的解压APP](https://play.google.com/store/apps/details?id=ru.zdevs.zarchiver)。

**优点：**
1.更省电(因为不像原始 Google Play服务 那样，频频被唤醒，平均每个装置可以省电至少15%以上，待机时间平均延长20%)。
2.隐私保障更安全(若 Google 要求装置回传资料，microG会尝试减少传送或是传送空白的假资料)。
3.安装包更小{microG Ver. 0.2.4-3-g47a61d6 预览版(20160731更新)，大小仅 2.12Mb 跟 Google Play 服务 Ver. 9.4.52(270-127739847)(20160731更新)大小 51.87 Mb ，两者空间占用率差了 24.4 倍，若是没有整合 Google Play 服务 的更新到系统，占用率差距至少差了 50 倍以上}
4.内存占用更少(若点击 microG 的主要应用程序，其应用只有一个服务 平均内存占用率仅 9Mb，若单纯被 Google Play商店 唤醒，也平均使用约5~6Mb，相反于Google Play 服务，一出现就先吃掉你 20Mb 的内存，服务还多达4~6个)。


**缺点:**
1.安装步骤稍微复杂(说穿了，其实只要搞懂安装顺序。其实困难度很低)
2.有时Play商店不会自动开始下载App(点开 microG 再回到 Play 商店 就又可以抓了，因为我有装绿色守护把它丢到黑名单，可能有时候唤醒不到吧)
3.有时候最下面的状态选项会无故无勾(其实不影响 Play商店 的下载)

就这样我的MicroG安装教程就到此结束了，感谢参与MicroG项目的人员同时还有这一篇文章[https://www.asus.com/zentalk/tw/forum.php?mod=viewthread&tid=162093](https://www.asus.com/zentalk/tw/forum.php?mod=viewthread&tid=162093)(也许需要科学上网)我就是从这一篇文章得知了MicroG项目，同时本文的内容也是基于这篇文章所诞生的。