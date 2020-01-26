---
layout: post
title: IPTV 与互联网融合
date: 2020-01-15 17:10
updated: 2020-01-20 00:46
categories: 教程
tags: 
    - IPTV
    - OpenWrt
    - Lean
    - udpxy
    - IGMP
    - RTSP
urlname: 23
comment: true
---

因为家里弱电箱到客厅电视只有一条线，在原有情况下无法做到 IPTV 和互联网在同一线路上通过。本教程不仅可以解决单线路同时播放互联网软件节目，还可以让任意设备播放 IPTV 节目。

<!-- more -->

## 前言

### 依赖

- 需要一个能够刷 OpenWrt 的路由器(需具有`数据包镜像`和 `udpxy` 功能/插件)用于抓包和后续的使用，因为其功耗较低且价格比较便宜 Newifi D2 拼多多 100 以下就可以拿下
- 能够正常播放节目的 IPTV 机顶盒，如果自己家里都没有能用的那就没有融合一说了

### 环境

> 因为不同地区网络环境不同，我这里是四川电信，但是可能在同一个省环境都会不一样，所以并不一定在其他地区可用。

| 设备 | 型号 | 软件 |
| -- | -- | -- |
| 光猫 | TEWA-500E | - |
| 主路由 | Newifi-D2 | Lean OpenWrt R9.6.1 |
| AP | 腾达 AC6 | - |

![网络结构拓扑图](https://st.blackyau.net/blog/23/1.png)

无需电信 IPTV 机顶盒，也可在任何设备上通过 http 链接直接播放直播节目。下图分别为 PotPlayer(PC) 和 超级直播(Android) 播放节目效果图。

![PC PotPlayer](https://st.blackyau.net/blog/23/2.jpg)

![Android 超级直播](https://st.blackyau.net/blog/23/3.jpg)

### iGMP与RTSP

在 IPTV 中常见的两种用于播放直播节目的协议分别为 IGMP 和 RTSP，他们之间的差异如下。

| 协议 | 节目类型 | 可用时间 | 鉴权 |
| :---: | :---: | :---: | :---: |
| IGMP | 直播 | 长期 | 强制 |
| RTSP | 直播/回放/点播 | 短期 | 非强制 |

#### IGMP

**网路群组管理协议**（英语：Internet Group Management Protocol，缩写：IGMP）是用于管理网路协议多播组成员的一种通信协议，有时候我也会将其称为**组播**。

在电信这边，组播地址通常很少变化，但是很重要的是它只能看直播不能看回放。又因为它的地址是内网地址，所以你必须要获取到电信的内网IP才能正常播放。我比较倾向于使用组播地址，因为电视节目回放有啥可看的，一般都是爱奇艺什么的了，而且最重要的是它的地址很少变化，这样就给不会倒腾的家人减少了很多麻烦。

原理上组播和广播(给网络里的所有人都发送一个消息)有点相似，但是组播会划分一个更小的范围，并且这个范围里面设备的名单会同时由客户端和主机端进行维护，路由器会根据不同的组别来转发不同的数据。

下面假装这个网络里面，有3组人正在分别在看3个节目。

![IGMP1](https://st.blackyau.net/blog/23/4.png)

正在收看 CCTV-251 的朋友说，这太假了。我想看点正能量的、让人血脉喷张的。然后请求换到 正在播放 CCTV-1 的 `igmp://239.93.22.133:9260`

![IGMP2](https://st.blackyau.net/blog/23/5.png)

路由器随后听到了这位朋友的呼唤，然后就将它放进了 CCTV-1 的组里。

![IGMP3](https://st.blackyau.net/blog/23/6.png)

通过上面的例子你大致能了解到 IGMP 协议的工作原理，可以简单的总结为 IGMP 就是 `一对多`，下面的一个例子则是和 IGMP 相反。

#### RTSP

**实时流协议**（Real Time Streaming Protocol，RTSP）是一种网络应用协议，专为娱乐和通信系统的使用，以控制流媒体服务器。该协议用于创建和控制终端之间的媒体会话。媒体服务器的客户端发布VCR命令，例如播放，录制和暂停，以便于实时控制从服务器到客户端（视频点播）或从客户端到服务器（语音录音）的媒体流，有时候我也会将其称为**时移**。

原理上 RTSP 和常见的 HTTP 协议比较相似，也就是 `一对一`，下面这个图可以帮助你理解。

你在观看节目的时候，可以随意的后退暂停，也可以自己想看什么就看什么，不用加入别人的组，整个资源都被你一个人享用。就和你平时看爱奇艺，B站什么的没区别。

![RTSP](https://st.blackyau.net/blog/23/7.png)

正如上面的介绍一样，RTSP 的主要特点就是可以时移，也就是可以拖动进度条。而且大部分地区的 RTSP 地址都是公网 IP，甚至还可以在获取到地址后，不需要任何授权都可以直接正常播放。

所以网络上流传的 IPTV 直播源基本都是 RTSP 地址。不过四川电信这边 RTSP 有鉴权，必须要以电信的内网 IP 访问才行。

同时又因为有部分套餐的 IPTV 是没有回放权限的，所以电信应该还需要验证是谁在播放，这就让观看它成为了比较麻烦的事情(不同地区情况不同，这里只针对我所在的地区)。

## 目标

通过了我上面对 IGMP 和 RTSP 协议的介绍，相信你对他俩都有了一定的了解。接下来我将为你详细介绍本次教程的内容。

- 适合本项目的硬件设备
- 软件安装及环境配置
- 抓包获取 IPTV 的 IGMP 和 RTSP 播放地址
- 使用 igmpproxy 将所有 IGMP 数据转发到 IPTV 口
- 使用 udpxy 将 IGMP 地址转换为 HTTP
- 在电视盒子、手机和 PC 上正常播放
- 在外欣赏自家 IPTV 直播源

你可以通过点击右侧边栏，来快速跳跃到你需要的章节或查阅你当前的浏览进度。

## 硬件

本章节会介绍你所需要的硬件设备，在抓包和使用融合网络时不可避免的会使用到，生活中平时少见的设备。

### 路由器

IPTV与互联网融合，的主要设备也就是路由器了。一款合适的路由器，可以和电脑一起直接走通整个教程。对于本教程而言，一个能够刷 `OpenWrt` 或 `Lede` 同时还能安装 `igmpproxy` 和 `udpxy` 最好还支持 `交换机端口镜像` 就完美了。当然也有一些教程是通过 `Padavan` 来实现的，但是我个人没有尝试过，这里就不做评价了。

![igmpproxy和udpxy 截图](https://st.blackyau.net/blog/23/8.jpg)

![交换机端口镜像 截图](https://st.blackyau.net/blog/23/9.jpg)

如果你目前没有具有此功能的路由器，我推荐你购买 新三 它还有其他的名字 新路由3、Newifi D2、Newifi 3、Newifi 3 D2。也都是同一款，得益于所谓的矿难，这款路由器目前淘宝、拼多多和转转之类的，100元以内都可以拿下，同时购买的时候推荐你加钱让卖家刷好 `Breed` 和我使用一样的硬件设备，这也能让你配置的时候少走一些弯路。

![新三 照片](https://st.blackyau.net/blog/23/10.jpg)

> Breed 相当于 Android 里面的 Recovery ，Windows 里面的 PE。可以让你在刷机的时候不会轻易翻车。

### 抓包工具

通过抓包获取 IPTV 的[组播地址](https://blackyau.cc/23.html#iGMP%E4%B8%8ERTSP)也是必不可少的一步。如果你的路由器没有 `交换机端口镜像` 的功能，你就需要淘宝单独购买一个 网络抓包工具 ，下图为 Amazon 搜索 `Throwing Star LAN Tap` 的外观图。

![Throwing Star LAN Tap](https://st.blackyau.net/blog/23/11.jpg)

动手能力比较强的朋友，也可以参考恩山无线论坛的这个帖子[小白的IPTV折腾教程（1）---0元DIY抓包神器](https://www.right.com.cn/FORUM/thread-328186-1-1.html)，利用两根网线和4个水晶头就可以做出一个具有同样功能的抓包工具。

不过我还是比较推荐刷一个具有 `交换机端口镜像` 功能的固件，毕竟直接就可以上手用。如果你是比较热门的机型，比如斐讯又或是我之前说到的 Newifi D2 或是其他搭载了 MT7621 芯片的路由器，应该都不难在恩山找到具有该功能的固件。

## 软件

下面列出了本次教程中所有需要的软件，我使用软件的版本，以及和下载链接。

| 设备 | 软件名 | 版本 | 下载地址 |
| -- | -- | -- | -- |
| PC | Wireshark | Portable 3.2.0 | [Wireshark 官网](https://www.wireshark.org/index.html#download) |
| PC | Notepad++ | 7.8.2 | [Notepad++ 官网](https://notepad-plus-plus.org/downloads/) |
| PC | Excel | Mondo 2016 x86 | [Office Tool Plus](https://otp.landian.vip/zh-cn/) |
| 路由器 | OpenWRT | R9.6.1 | [橙子的个人博客](https://www.maxlicheng.com/openwrt/225.html) |
| 路由器 | igmpproxy | 0.2.1-4 | 固件自带 |
| 路由器 | udpxy | 2016-09-18-53e4672a75..4-1 | 固件自带 |
| 路由器 | luci-app-udpxy | git-19.146.62144-fd6fdb2-1 | 固件自带 |

下面介绍了每个软件的用途。

| 软件名 | 用途 |
| -- | -- |
| Wireshark | 抓包获取 IPTV 播放地址 |
| Notepad++ | 抓包后数据整理 |
| Excel | 抓包后数据整理 |
| OpenWRT | 路由器固件 |
| igmpproxy | 转发IGMP流量到指定端口 |
| udpxy | IGMP流量转HTTP |

## 抓包

首先将来自光猫的互联网和往常一样连接到路由器的 WAN 口，将 ITV 口连接到路由器的 LAN 4 口，将 IPTV 盒子连接到路由器的 LAN 3 口，最后将 LAN 1 口连接至电脑。

![抓包连线](https://st.blackyau.net/blog/23/12.png)

随后配置路由器的流量镜像功能，将接有 IPTV 盒子的 LAN 3 口设置为 数据包镜像源端口 ,将接有电脑的 LAN 1 口设置为 数据包镜像监听端口。其他 VLAN 设置无需改动。

![数据镜像设置](https://st.blackyau.net/blog/23/13.jpg)

保存并应用设置后，即可在电脑上启动 Wireshark 并监听以太网接口，随后启动 IPTV 盒子。

启动 IPTV 盒子后，应不停的有数据显示在窗口中。然后使用 IPTV 盒子遥控器，进入直播随便播放一个节目，等节目可以正常播放的时候，即可在 Wireshark 中停止捕获。同时也可关闭 IPTV 盒子，接下来就是分析数据了。

> 如果你没有看到任何数据跳动，或者是特别少，应注意是否端口插错，或者是在设置流量镜像的地方有错

## 分析抓包数据

**因为不同地区的数据样式差异较大，我这里是四川电信，其他地区可供参考**

### 获取地址

在过滤器栏输入

```
http.request.uri contains "frameset_builder.jsp"
```

右键第一个请求，追踪流 - HTTP 流

![Wireshark 1](https://st.blackyau.net/blog/23/14.jpg)

再弹出的新窗口中查找

```
igmp://
```

如果你找到了类似下图 `igmp://239.93.22.133:9260` 的连接，那么恭喜你，你的数据抓包已经完成了最重要的定位了！(不难发现旁边也获取到了 `rtsp://` 开头的时移地址)

![Wireshark 2](https://st.blackyau.net/blog/23/15.jpg)

单击连接，主窗口就会自动定位到该请求。

![Wireshark 3](https://st.blackyau.net/blog/23/16.jpg)

单击展开该请求的完整内容，查看里面的内容是不是含有 `igmp://` 之类的重要数据。

![Wireshark 4](https://st.blackyau.net/blog/23/17.jpg)

右键 Line-based text data 导出分组字节流，随便取个名字保存到你能找到的地方。

![Wireshark 5](https://st.blackyau.net/blog/23/18.jpg)

用 notepad++ 打开，查看是否显示正常(前几行都是回车，会一片空白往后滑一点)。

![Wireshark 6](https://st.blackyau.net/blog/23/19.jpg)

### 无法获取到地址

如果你在窗口中一个数据都未获取到，那么请检查数据镜像设置或网线位置是否有错。

如果你是四川省，请仔细检查是不是，在过滤的时候复制错了或漏了内容。

如果你非四川省，可以在过滤器中输入

```
http
```

进行检索，一条一条的看，里面总会有 `igmp://` 之类获取节目单的数据(我也是这样找出来的)，如果找不到你也可以向我求助 Gmail:blackyau426@gmail.com

### 格式化数据

开始格式化之前，建议保存好原始文件。

替换完毕后可以将名字带有 PIP 的删除，这是用于机顶盒画中画功能的，说白了就是降低了分辨率的，我们就留下正常的和高清的就行了。

#### M3U8

此格式文件可以在 PC 中直接使用 VLC media player 和 PotPlayer 打开并播放

查找目标

```
.*ChannelName="(.*)",UserChannelID="(.*)",ChannelURL="igmp://(.*)",TimeShift=.*
```

替换为

```
#EXTINF:-1, \1\r\nhttp://192.168.10.1:8888/udp/\3
```

![M3U8 替换中](https://st.blackyau.net/blog/23/20.jpg)

将文档中被格式化了的数据，复制到新文档，并在文档首行写入

```
#EXTM3U
```

处理完毕后效果如图

![M3U8 替换完毕](https://st.blackyau.net/blog/23/21.jpg)

#### 超级直播

此文件可以在 Android 端的超级直播使用，电脑打开 在软件里面按返回时提示的网址，可以将自定义源上传至该软件。

查找目标

```
.*ChannelName="(.*)",UserChannelID="(.*)",ChannelURL="igmp://(.*)",TimeShift=.*
```

替换为

```
\1,http://192.168.10.1:8888/udp/\3
```

再次查找目标，删除空白行

```
\n\s*\r
```

替换为空白

全选改文本所有内容后复制，在顶部 编码 - 编码字符集 - 中文 - GB2312，确认切换到该字符集。然后删除文本所有内容，并粘贴。最后将文件保存为txt即可。

> 超级直播中文本编码格式必须为 GB2312 否则中文会乱码

![修改编码字符集](https://st.blackyau.net/blog/23/22.jpg)

处理完毕后效果如图

![超级直播效果图](https://st.blackyau.net/blog/23/23.jpg)

## 获取IPTV内网地址

