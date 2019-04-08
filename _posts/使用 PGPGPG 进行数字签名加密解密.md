---
layout: post
title: 使用 PGP/GPG 进行数字签名加密解密
date: 2018-06-09 23:27:00
updated: 2019-04-06 11:36:24
categories: 教程
tags: 
    - PGP
    - GPG
    - 数字签名
    - 加密
    - 解密
urlname: 9
comment: true
---
## PGP/GPG/GnuPG/OpenPGP 关系和区别

PGP 是 Phil Zimmermann 于1991年开发的商业应用程序，1997年7月，PGP Inc.与齐默尔曼同意 IETF 制定一项公开的互联网标准，称作 OpenPGP，任何支持这一标准的程序也被允许称作 OpenPGP。GnuPG 则是支持了这一标准的开源应用程序。提个小故事：

<!-- more -->

> 当年的 PGP 之父（Phil Zimmermann）因为发明了 PGP 这款文件/邮件加密工具，差点被美国政府抓去坐牢。当时（上世纪90年代初）的美国法律规定，“高强度加密技术”属于军用技术，不允许出口到海外。Phil Zimmermann 后来想了个妙计——通过 MIT 出版社出了一本书，把 PGP 的全部源代码放到书中，然后援引“美国宪法第一修正案”（出版物受“言论自由”的保护），才推翻了对他的指控。[<sup>来源</sup>](https://program-think.blogspot.com/2014/06/truecrypt-dead.html)

## PGP 加解密和数字签名简单介绍

我们可以生成世界上唯一的一组公钥和密钥，公钥用于加密，密钥用于解密。我们一般只公开公钥，他人使用公钥加密后在通过任何方式发送给我。但是因为密钥只能解密*使用与自己配对的公钥加密后的数据*，这样就保证了使用了我公钥加密后的数据只能由我解密查看。这就是PGP加解密的基本原理。

数字签名则是用来校验文件完整性的它不具备加密功能，它在发送的信息末尾都会加上一个用于检验的字段（签名）。这个签名是基于发布者的信息和文件计算出来的，接收者使用发送者的公钥就可以核对信息是否遭到篡改。大部分支持OpenPGP协议的应用，都会在加密的同时默认开启这个特性。

## Windows 下使用 GnuPG 加密/解密/签名/校验

Windows 环境下我选择 [Gpg4win](https://gpg4win.org) 作为本次教程的对象，它使用 GnuPG 作为加解密后端、使用 Kleopatra 作为证书管理器和通用加密对话框。更多可查看[关于](https://gpg4win.org/about.html)，并且它拥有可视化图形界面，最重要的是它以及它包含的工具都是[开源应用](https://gpg4win.org/about.html)。

### 下载安装 Gpg4win

直接在 Gpg4win 主页 https://gpg4win.org 下载并安装到你认为合适的路径即可

### 使用 Gpg4win 加解密/签名检验文件

打开桌面或开始中的 Kleopatra ，并点击「新建密钥对」

![1](https://st.blackyau.net/blog/9/1.png)

在对话框中输入你所期望的信息(对任何人都是可见的)

![2](https://st.blackyau.net/blog/9/2.png)

会让你输入一个密码，用来加密密钥

![3](https://st.blackyau.net/blog/9/3.png)

「生成你的密钥对的副本」可以让你保存密钥和公钥到其他地方，「将公钥上传到目录服务」可以将你的公钥上传到服务器，这样的话别人就可以通过昵称/邮箱或指纹搜索到你的公钥(只有指纹具有唯一性)。注：将公钥上传到服务器是一种不可逆的行为，就算密钥到期了也不会被删除而是会被标记上「已过期」。

Kleopatra默认密钥服务器地址:[http://pool.sks-keyservers.net/](http://pool.sks-keyservers.net/)

![4](https://st.blackyau.net/blog/9/4.png)

新建密钥对后就可以选择加密文件了，点击左上角的「签名/加密」。选择一个文件加密后，别人在没有你的私钥的情况下就无法解密你的文件了。当你需要使用别人的公钥加密文件时，要先导入他人的公钥。并在加密文件时勾选「为他人加密」，在输入他人的名字并选中即可。

![5](https://st.blackyau.net/blog/9/5.png)

解密时你必须要先导入对应的密钥对，同时还需要提供密码，并点击「Save All」

![6](https://st.blackyau.net/blog/9/6.png)

### 使用 Gpg4win 加解密/签名检验文本

点击工具栏最后一项「记事本」，可以对纯文本进行加解密和签名校验。

![7](https://st.blackyau.net/blog/9/7.png)

在文本框中填入内容在「收件人」选项中选择加解密或签名检验密钥，点击上方的「签名/加密 Notepa」或「Decryep /Verify Notepa」可进行相应的操作。

![8](https://st.blackyau.net/blog/9/8.png)

### 导出密钥对

选中你的密钥，右键「导出绝密密钥」即可

## 来试试使用PGP解密以下纯文本吧

密文:https://st.blackyau.net/blog/file/Ciphertext.txt (你需要将里面的内容复制出来)

密钥对:https://st.blackyau.net/blog/file/LetPGPFly.gpg (可加密/解密)

公钥:https://st.blackyau.net/blog/file/LetPGPFly.asc (仅可加密)

## 其他平台
Android: OpenKeychain: Easy PGP(加密时第一项「加密到」需要你手动输入加密密钥的昵称)

> Google Play:https://play.google.com/store/apps/details?id=org.sufficientlysecure.keychain
> F-Droid:https://f-droid.org/packages/org.sufficientlysecure.keychain/
> [LetITFly BBS@Yongmeng使用 OpenKeychain 管理 OpenPGP 密钥](https://bbs.letitfly.me/d/985)

Mac: GPG Suite

> https://gpgtools.org/

Linux: GnuPG

> https://www.gnupg.org/download/index.html

## 外部链接

[分析一下 TrueCrypt 之死（自杀 or 他杀？），介绍一下应对措施 - 编程随想](https://program-think.blogspot.com/2014/06/truecrypt-dead.html#head-2)

[扫盲文件完整性校验——关于散列值和数字签名 - 编程随想](https://program-think.blogspot.com/2013/02/file-integrity-check.html#head-9)

[为什么桌面系统装 Linux 可以做到更好的安全性（相比 Windows & macOS 而言） - 编程随想](https://program-think.blogspot.com/2017/03/Why-Linux-Is-More-Secure-Than-Windows-and-macOS.html#head-7)

[上传到公钥服务器的gpg公钥过期了会被删除吗？ 知乎@胡涵铭](https://www.zhihu.com/question/60520344/answer/218561457)

[使用GnuPG(PGP)加密信息及数字签名教程 - 月光博客](http://www.williamlong.info/archives/3439.html)

## 版权声明

本文中除引用自互联网内容外，其他内容均以[CC0 1.0 通用 (CC0 1.0) 公共领域贡献](https://creativecommons.org/publicdomain/zero/1.0/deed.zh)方式授权。我已经将作品 贡献 至公共领域，在法律允许的范围，放弃所有我在全世界范围内基于著作权法对作品享有的权利，包括所有相关权利和邻接权利。 您可以复制、修改、发行和表演本作品，甚至可用于商业性目的，都无需同意和标出署名。

![license](https://licensebuttons.net/p/zero/1.0/88x15.png)