---
layout: post
title: 已实现全站HTTPS安全连接
date: 2016-08-14 11:36:00
updated: 2018-06-24 23:50:25
categories: 教程
tags: 
    - 教程
    - WEB
urlname: 2
comment: true
---
**现在blog使用的VPS搭建，与这个实现方法有较大的差异。会更新的**

高三开学的第一天，这个特殊的日子。本站终于实现了全站完美HTTPS加密访问，在这个人人都注重隐私的时代。我这个博客当然不能落后于别人，顺便也在这里记录一下添加HTTPS访问时遇到的问题。

<!-- more -->

首先，需要一个[SSL证书](http://baike.baidu.com/view/1009264.htm)。我的证书是使用的[沃通DV免费SSL证书](https://freessl.wosign.com)，毕竟一个穷逼学生党没多少钱就用的免费的..
其次，因为本站是搭建于虚拟主机之上。所以导入证书和私钥都很简单，直接用Notepad++打开，把里面的内容粘贴进面板对应SSL设置的地方就行。

满怀信心在地址栏中输入了连接，打开后却没有**高贵的绿色小锁**。使用F12检查后，发现是[我选择的主题](https://github.com/newraina/typecho-theme-NexTPisces)中的 `header.php` 引用了Google前端公共库没有通过HTTPS传输。而且更为捉急是，我来回更换了好几个国内的CDN库全都不支持HTTPS。当然了这种问题是阻挡不了绿色小锁对我的诱惑。

我直接将他引用的CSS，全部下载到本地修改后上传到主机。然后把header.php中引用的地址改为了主机。就这样我的博客前面就有了耀眼的绿色小锁。同时还将站点中的静态文件全部放到了[七牛云储存](https://www.qiniu.com/)中，七牛云支持HTTPS链接还是挺不错的！最主要的是他每月有免费的10G流量，这对于我的博客绰绰有余。

对于上传的图片如何使用七牛云托管，我也通过google找到了[解决方法](https://www.xiaoz.me/archives/6842)。设置镜像储存的镜像源为我的博客
![img](https://st.blackyau.net/blog/3/1.jpg)
然后修改post.php中

```php
    <?php $this->content(); ?>
```

其中的 https://blackyau.cc 为我的博客域名，https://obvrtdse2.qnssl.com 为七牛云空间的域名，这个方法的原理就是七牛支持镜像存储，设置镜像源后，当你访问七牛的地址会自动从源地址获取对应文件并抓取过来，连SDK都不需要使用。

经过了这样一鼓捣，开学了就可以安心的写字了。
![img](https://st.blackyau.net/blog/3/2.png)