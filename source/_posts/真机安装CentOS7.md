---
layout: post
title: 真机安装 CentOS 7
date: 2019-03-27 19:45:17
updated: 2025-03-02 20:55:01
categories: 教程
tags: 
    - Linux
    - CentOS
urlname: 10
comment: true
---

在家里用一台老古董台式机装了个 CentOS 遇到了一些以前用云服务器根本不会遇到的问题，这里也就做一下记录。

{% note danger %}
#### 警告
CentOS Linux 7 已经于 2024 年 6 月 30 日终止生命周期（EOL），不建议继续在继续使用 CentOS 7 了，推荐使用 [Rocky Linux](https://rockylinux.org/) 替代。
{% endnote %}

<!-- more -->

## 准备工作

直接在 [官网](https://www.centos.org/download/) 下载的完整镜像，然后用 UltraISO 写进U盘制作成启动盘的。其实在安装 CentOS 之前，我为了安装 Ubuntu 断断续续折腾了两天的时间。就是初始化安装完毕后，始终进不去系统实在是搞不明白所以就转投了 CentOS 。

## 进入安装向导

使用 ```UEFI``` 方式从U盘启动后，会进入到这个界面。👇

![1](https://st.blackyau.net/blog/10/1.png)

按 ```e``` 编辑一下(如果是使用 ```Legacy BIOS``` 启动，则按 ```tab``` 键)，将第一行的 ```linuxefi /images/pxeboot/vmlinuz inst.stage2=hd:.......``` 👇

![2](https://st.blackyau.net/blog/10/2.png)

改为下面的 ```linuxefi /images/pxeboot/vmlinuz initrd=initrd.img linux dd quiet``` 这里是为了确定启动盘的 ```设备名``` 便于后续的安装。👇

![3](https://st.blackyau.net/blog/10/3.png)

按下 ```Ctrl-x``` 后，等待片刻你就会在这里停下来。最好在这里拍个照记录一下，自己通过 ```LABEL``` 和 ```TYPE``` 记住 CentOS 启动盘的 ```设备名``` 。（下图是 ```sda4``` ）👇

![4](https://st.blackyau.net/blog/10/4.png)

按电源键重启计算机，还是从U盘启动。还是按 ```e``` 编辑一下，不过这次要把第一行修改为 ```linuxefi /images/pxeboot/vmlinuz inst.stage2=hd:/dev/设备名 quiet``` 。（下图是 ```sda4``` ）然后按下```Ctrl+x``` 进入安装向导。

![5](https://st.blackyau.net/blog/10/5.png)

![6](https://st.blackyau.net/blog/10/6.png)

## 安装向导

### 选择语言

毫不犹豫的选择 ```English``` ，遇到了一些问题比较方便找到解决方法。

![7](https://st.blackyau.net/blog/10/7.png)

### 硬盘分区

点击 ```SYSTEM - INSTALLATION DESTINATION``` 

![8](https://st.blackyau.net/blog/10/8.png)

选中要作为启动盘的硬盘，并点击左下方的 ```Set as Boot Device``` 

![9](https://st.blackyau.net/blog/10/9.jpg)

因为我有多个物理磁盘，这里我使用了手动分配。并使用 LVM 进行磁盘管理，方便后续的扩容。

点击下面的 ```x storage devices selected``` 勾选上所有磁盘，在进行下一步的处理。

![10](https://st.blackyau.net/blog/10/10.jpg)

添加分区后，点击左侧 ```Volume Group - Modify``` 创建一个名为 ```all``` 分组，并选择了所有的磁盘。再逐步调整。

我的设置如下：

![11](https://st.blackyau.net/blog/10/11.jpg)

![12](https://st.blackyau.net/blog/10/12.jpg)

还需要注意一下， ```bios``` 不能使用 ```LVM``` 分组。文件系统也不能使用 ```LVM``` ，这里的 ```/``` 和 ```swap``` 分区大小请各位根据自己的情况而定。

### 时间

磁盘配置完毕后，建议配置一下时间。建议与计算机当前当地时间保持一致，或者是你期望的位置。

![13](https://st.blackyau.net/blog/10/13.jpg)


### 登陆凭证

一切准备就绪后，点击右下方的 ```Begin Installation``` 即可开始最后的登陆凭证设置。这里就不再赘述。

![14](https://st.blackyau.net/blog/10/14.png)

## 安装完毕

![15](https://st.blackyau.net/blog/10/15.png)

## 参考:

[鳥哥的 Linux 私房菜 - 第三章、安裝 CentOS7.x](http://linux.vbird.org/linux_basic/0157installcentos7.php)

[知乎@cenbug - 真机安装CentOS7/Linux](https://zhuanlan.zhihu.com/p/35161351)

[博客园@期待某一天 - 笔记本真机安装centos7](https://www.cnblogs.com/wudongyu/p/6673784.html)