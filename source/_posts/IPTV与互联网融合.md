---
layout: post
title: IPTV 与互联网融合
date: 2020-01-15 17:10
updated: 2020-04-29 11:44
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

### iGMP 与 RTSP

在 IPTV 中常见的两种用于播放直播节目的协议分别为 IGMP 和 RTSP，他们之间的差异如下。

| 协议 | 节目类型 | 可用时间 | 鉴权 |
| :---: | :---: | :---: | :---: |
| IGMP | 直播 | 长期 | 强制 |
| RTSP | 直播/回放/点播 | 短期 | 非强制 |

#### IGMP

**网路群组管理协议**（英语：Internet Group Management Protocol，缩写：IGMP）是用于管理网路协议多播组成员的一种通信协议，有时候我也会将其称为**组播**。

在电信这边，组播地址通常很少变化，但是很重要的是它只能看直播不能看回放。又因为它的地址是内网地址，所以你必须要获取到电信的内网IP才能正常播放。我比较倾向于使用组播地址，因为电视节目回放有啥可看的，一般都是爱奇艺什么的了，而且最重要的是它的地址很少变化，这样就给不会倒腾的家人减少了很多麻烦。

原理上组播和广播(给网络里的所有人都发送一个消息)有点相似，但是组播会划分一个更小的范围，并且这个范围里面设备的名单会同时由客户端和主机端进行维护，路由器会根据不同的组别来转发不同的数据。

下面假装这个网络里面，有 3组 人正在分别在看 3个 节目。

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

> Breed 相当于 Android 里面的 Recovery，Windows 里面的 PE。可以让你在刷机的时候不会轻易翻车。

### 抓包工具

通过抓包获取 IPTV 的[组播地址](https://blackyau.cc/23.html#iGMP-%E4%B8%8E-RTSP)也是必不可少的一步。如果你的路由器没有 `交换机端口镜像` 的功能，你就需要淘宝单独购买一个 网络抓包工具，下图为 Amazon 搜索 `Throwing Star LAN Tap` 的外观图。

![Throwing Star LAN Tap](https://st.blackyau.net/blog/23/11.jpg)

动手能力比较强的朋友，也可以参考恩山无线论坛的这个帖子[小白的IPTV折腾教程（1）---0元DIY抓包神器](https://www.right.com.cn/FORUM/thread-328186-1-1.html)，利用两根网线和4个水晶头就可以做出一个具有同样功能的抓包工具。

不过我还是比较推荐刷一个具有 `交换机端口镜像` 功能的固件，毕竟直接就可以上手用。如果你是比较热门的机型，比如斐讯又或是我之前说到的 Newifi D2 或是其他搭载了 MT7621 芯片的路由器，应该都不难在恩山找到具有该功能的固件。

## 软件

下面列出了本次教程中所有需要的软件，我使用软件的版本，以及和下载链接。

| 设备 | 软件名 | 版本 | 下载地址 |
| -- | -- | -- | -- |
| PC | Wireshark | Portable 3.2.0 | [Wireshark 官网](https://www.wireshark.org/index.html#download) |
| PC | Notepad++ | 7.8.2 | [Notepad++ 官网](https://notepad-plus-plus.org/downloads/) |
| PC | Xshell | 6.0.0032 | [Netsarang 家庭/学校版](https://www.netsarang.com/zh/free-for-home-school/) |
| 路由器 | OpenWRT | R20.2.15 | [newifi-d2.zip](https://st.blackyau.net/blog/23/newifi-d2.zip) |
| 路由器 | igmpproxy | 0.2.1-4 | 固件自带 |
| 路由器 | udpxy | 2016-09-18-53e4672a75..4-1 | 固件自带 |
| 路由器 | luci-app-udpxy | git-19.146.62144-fd6fdb2-1 | 固件自带 |

下面介绍了每个软件的用途。

| 软件名 | 用途 |
| -- | -- |
| Wireshark | 抓包获取 IPTV 播放地址 |
| Notepad++ | 抓包后数据整理 |
| Xshell | SSH连接路由器 |
| OpenWRT | 路由器固件 |
| igmpproxy | 转发IGMP流量到指定端口 |
| udpxy | IGMP流量转HTTP |

这里还有个友华 WR1200JS 可用的固件：[youhua_wr1200js.zip](https://st.blackyau.net/blog/23/youhua_wr1200js.zip)

## 抓包

首先将来自光猫的互联网和往常一样连接到路由器的 WAN 口，将 ITV 口连接到路由器的 LAN 4 口，将 IPTV 盒子连接到路由器的 LAN 3 口，最后将 LAN 1 口连接至电脑。

![抓包连线](https://st.blackyau.net/blog/23/12.png)

随后配置路由器的流量镜像功能（我提供的固件中路由器 IP 为 192.168.1.1），将接有 IPTV 盒子的 LAN 3 口设置为 数据包镜像源端口，将接有电脑的 LAN 1 口设置为 数据包镜像监听端口。其他 VLAN 设置无需改动。

![数据镜像设置](https://st.blackyau.net/blog/23/13.jpg)

保存并应用设置后，请严格按照以下步骤进行抓包：

- 确认已经关闭电脑中所有程序，也包括 Wireshark
- 切断 IPTV 盒子电源
- 启动 Wireshark 并监听以太网接口
- 接通 IPTV 盒子电源
- 使用 IPTV 盒子遥控器，选择直播，并播放一个节目
- 让节目保持正常播放 2~3 秒
- 停止 Wireshark 抓包并保存抓包数据
- 切断 IPTV 盒子电源

要关闭电脑中的其他联网程序，不然它也会被抓进来，干扰分析。

关闭 Wireshark 是方便定位 IPTV 盒子的初始化信息。让 IPTV 盒子初始化的信息，就在最初的几个数据包里面。

切断 IPTV 盒子电源并重开，是为了能够抓到有关初始化的信息，比如获取 IP 的方法，是 DHCP 获取 IP，还是 PPPOE 之类的。

启动 IPTV 盒子后，应不停的有数据显示在窗口中。

> 如果你没有看到任何数据跳动，或者是特别少，应注意是否端口插错，或者是在设置流量镜像的地方有错。

- 播放节目是为了让 IPTV 盒子去拉取直播节目列表，我们最需要的也就是直播节目列表信息。

## 分析抓包数据

**因为不同地区的数据样式差异较大，我这里是四川电信，其他地区可供参考**

### 获取地址

在过滤器栏输入

```
http.request.uri contains "frameset_builder.jsp"
```

四川成都电信可以尝试

```
http.request.uri contains "AuthenticationURL"
```

右键第一个请求，追踪流 - HTTP 流

![Wireshark 1](https://st.blackyau.net/blog/23/14.jpg)

再弹出的新窗口中查找

```
igmp://
```

如果你找到了类似下图 `igmp://239.93.22.133:9260` 的连接，那么恭喜你，你的数据抓包已经完成了最重要的定位了！（不难发现旁边也获取到了 `rtsp://` 开头的时移地址）

![Wireshark 2](https://st.blackyau.net/blog/23/15.jpg)

单击链接，主窗口就会自动定位到该请求。

![Wireshark 3](https://st.blackyau.net/blog/23/16.jpg)

单击展开该请求的完整内容，查看里面的内容是不是含有 `igmp://` 之类的重要数据。

![Wireshark 4](https://st.blackyau.net/blog/23/17.jpg)

右键 Line-based text data 导出分组字节流，随便取个名字保存到你能找到的地方。

![Wireshark 5](https://st.blackyau.net/blog/23/18.jpg)

用 notepad++ 打开，查看是否显示正常(前几行都是回车，会一片空白，往后滑一点就能看到了)。

![Wireshark 6](https://st.blackyau.net/blog/23/19.jpg)

### 无法获取到地址

如果你在窗口中一个数据都未获取到，那么请检查数据镜像设置或网线位置是否有错。

如果你是四川省，请仔细检查是不是，在过滤的时候复制错了或漏了内容。

如果你非四川省，可以在过滤器中输入

```
http
```

进行检索，一条一条的看，里面总会有 `igmp://` 之类获取节目单的数据（我也是这样找出来的），如果找不到你也可以向我求助 Gmail:blackyau426@gmail.com

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

处理完毕后效果如图，最后将文件另存为 .m3u8 即可。

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

全选该文本所有内容后复制，在顶部 编码 - 编码字符集 - 中文 - GB2312，确认切换到该字符集。然后删除文本所有内容，并粘贴。最后将文件保存为 txt 即可。

> 超级直播中文本编码格式必须为 GB2312 否则中文会乱码

![修改编码字符集](https://st.blackyau.net/blog/23/22.jpg)

处理完毕后效果如图

![超级直播效果图](https://st.blackyau.net/blog/23/23.jpg)

## 获取 IPTV 内网地址

四川电信是 DHCP 获取，我在网上看很多地方都是 PPPOE 所以用户名和密码你们就需要自己翻翻 IPTV 盒子的设置拉~

这边也会使用到抓包的数据，应该就是前几个了，找到 `Dynamic Host Configuration Protocol (Request)` 请求，展开 `Option: (60) Vendor class identifier` 和`Option: (12) Host Name` 以及 `Client MAC address` 。都需要右键 - 复制 - 值 。

![DHCP](https://st.blackyau.net/blog/23/24.jpg)

如果你没有找到 DHCP 的数据包，可以通过 IPTV 盒子底部的贴纸查看。我这款盒子，最后一个就是 `Option: (12) Host Name` 当然了 MAC 地址上面也有，而 `Option: (60) Vendor class identifier` 我已经在上图给你了，就是 `SCITV` 。

![IPTV盒子](https://st.blackyau.net/blog/23/25.jpg)

接下来就开始路由器的设置了。

首先进入路由器设置 - 网络 - 交换机，将之前用于抓包的 数据包镜像 功能关掉。随后将插有 ITV 口的 LAN 4 在 VLAN 1 中设置为 `关` 。添加一个 VLAN 3 ，将 CPU (eth0) 设置为 `tagged` ，然后将 VLAN 3 的 LAN 4 设置为 `untagged` 。设置完毕后，效果如下图。

![VLAN](https://st.blackyau.net/blog/23/26.jpg)

进入路由器设置 - 网络 - 接口 - 添加新接口。命名为 `IPTV` 注意全部大写，接口协议为 `DHCP 客户端` 包括接口 `VLAN:eth0.3` 。设置如下图。

![添加新接口](https://st.blackyau.net/blog/23/27.jpg)

然后设置端口的 请求 DHCP 时发送的主机名 对应的就是之前获取的 `Option: (12) Host Name`，以及高级设置里面的 请求 DHCP 时发送的 Vendor Class 选项 也就是之前获取的 `Option: (60) Vendor class identifier` 即 `SCITV`，最后是 重设 MAC 地址 填入 `Client MAC address` 也就是你 IPTV 盒子的 MAC 地址。

还有不要勾选 使用内置的 IPv6 管理，使用网关跃点为 20 。

![IPTV 接口设置](https://st.blackyau.net/blog/23/28.jpg)

保存并应用设置后，再进入你的 WAN 接口设置，将它的 网络跃点设置为 10，**否则你会无法正常使用互联网**。

进行到这里，你的 IPTV 接口应该就可以正常的获取到 `10` 开头的内网 IP 了。如果你不是四川的朋友，那么你地区的运营商可能是 PPPOE 验证或验证逻辑与我这里不同，如果你是四川的朋友，那么请你检查 之前抓包或者是在机顶盒上面看到的 `Option: (12) Host Name` 以及 `Option: (60) Vendor class identifier` 和 MAC 地址是否填写正确。如果你是通过抓包获取的数据，那应该不会有错，如果你是抄的机顶盒上面的，那么可能是因为你所在的地区与我的验证逻辑不同。

所以我强烈建议，还是通过抓包来分析 IPTV 盒子获得内网 IP 的全过程，因为不管你是 PPPOE 还是 DHCP 它都可以分析出来。

## 配置 igmpproxy 和 udpxy

### 使用 SSH 连接到路由器

修改配置文件时需要使用 SSH 连接到路由器进行修改，进入路由器设置 - 系统 - 管理权，在接口 lan 下设置端口为 22，同时打开 密码验证和允许 root 用户凭密码登录。

![路由器SSH设置](https://st.blackyau.net/blog/23/29.jpg)

下载 Xshell https://www.netsarang.com/zh/free-for-home-school/ ，官网提供了免费的供家庭和学校使用的版本，足够本次教程所用。

新建连接，名称随意，主机填上路由器的 IP。点击左侧连接中的用户身份验证，将方法设置为 Password 用户名为 root 密码则为登录 Web 端后台时的密码，我提供的固件默认是 `password`。

![Xshell](https://st.blackyau.net/blog/23/30.jpg)

### 安装 igmpproxy 和 udpxy

> 如果你的路由器使用的我提供的固件则无需安装，因为固件是自带该软件包的。

我建议在安装之前，在 Web 端后台的系统 - 备份/升级 中备份当前配置文件。因为我尝试了多个固件，在安装了 udpxy 后 Web 端就会无法正常使用，有很多报错。只有恢复到出厂设置才恢复正常。最后找到了一个自带 udpxy 的固件才解决我的问题。

使用 Xshell 连接到路由器后执行以下命令。

```shell
opkg update && opkg install igmpproxy udpxy luci-app-udpxy
```

`opkg update` 是用来更新软件列表的，因为大陆对 OpenWrt 软件源地址连通性不佳，所以可能需要等很久或者是多次尝试。

查看命令返回的结果或查看系统 - 软件包中的已安装软件包中是否存在 `igmpproxy` `udpxy` `luci-app-udpxy` 来判断是否安装成功。

### 配置 igmpproxy

关于 igmpproxy 它主要是将所有来自 lan 的 IGMP 数据都传到 IPTV 接口去，为了防止组播的 udp 数据在 lan 里面乱串，影响网络效率。但是我这里在 lan 里面是无法播放 `igmp://` 地址的数据的，我也不清楚是什么情况。而且从 [lede issues](https://github.com/coolsnowwolf/lede/issues/2841) 可知 lede 的 igmpproxy 是失效的，如果有人在 lan 里面观看组播地址视频或者是使用 IPTV 盒子，都会导致局域网内的组播风暴，会导致网络堵塞。所以主要是后面的 udpxy 在起作用，**你完全可以不配置 igmpproxy 使用 http 地址播放依然是可行的**。

执行以下命令，一定要复制全一起粘贴进去然后再回车执行。

```shell
echo "config igmpproxy
	option quickleave 1

config phyint
	option network IPTV
	option direction upstream
	list altnet 0.0.0.0/0

config phyint
	option network lan
	option zone lan
	option direction downstream" > /etc/config/igmpproxy
```

### 配置 udpxy

在路由器 Web 端设置 - 服务 - udpxy 中，勾选启动、Respawn、状态。将端口设置为 `8888`，将 Source IP/Interface 设置为 IPTV 接口的 ifname，也就是在路由器 Web 端设置 - 网络 - 接口 中 IPTV 接口图标下方的小字。在我这里为 `eth0.3` 。

![udpxy](https://st.blackyau.net/blog/23/31.jpg)

如果你有多设备同时播放的需求，那么请根据情况设置 `Max clients` 选项的值，它可以控制同时播放的终端数，该值默认为 3 ，最大可为 5000 。

> 感谢 [@xujuntc](https://www.right.com.cn/forum/thread-2921260-2-1.html) 的提醒。

udpxy 配置项介绍

| 配置项 | 配置文件 | 中文 | 说明 | 默认 |
| -- | -- | -- | -- | -- |
| disabled | 无 | 启动 | 启动 udpxy | 关闭 |
| Respawn | respawn | 重启 | 允许在 status 中重启 udpxy | 关闭 |
| verbose | verbose | 详细 | 启用详细日志输出 | 关闭 | 关闭 |
| status | status | 状态 | 启用 Web 端统计信息 | 关闭 |
| Bind IP/Interface | source | 监听地址/接口 | 要监听的地址或接口 | 0.0.0.0 |
| Port | port | 端口 | 监听端口 | 必填 |
| Source IP/Interface | source | 源 IP/端口 | 组播数据源的 IP 或端口 | 0.0.0.0 |
| Max clients | max_clients | 最大客户端数 | 同时播放的终端数 | 3(最大可设为5000) |
| Log file | log_file | 日志目录 | 日志输出的目录 | stderr(即打印在终端) |
| Buffer size | buffer_size | 缓冲大小 | 组播数据入站的缓冲区大小 | 2048 bytes(可选 `65536`, `32Kb`, `1Mb`) |
| Buffer messages | buffer_messages | 缓冲信息 | 向组播组请求多少数据并储存起来(单位:秒) | 1 |
| Buffer time | buffer_time | 缓存保存时间 | 数据可在缓冲区内保存的最长时间 | 1 |
| Nice increment | nice_increment | 未知 | 未知 | 0 |
| Multicast subscription renew | mcsub_renew | 定期重新加入组播组 | 每隔一段时间重新加入组播组,防止网络波动导致丢失组播连接(单位:秒) | 0 |

> 在路由器设置的 Web 端不知为什么后面 4 个选项，保存时都说值有误。如果有需要修改的朋友，只有手动修改 `/etc/config/udpxy` 配置文件了。下面是我的配置，可供参考。

```config
config udpxy
        option respawn '1'
        option verbose '0'
        option status '1'
        option disabled '0'
        option port '8888'
        option source 'eth0.3'
        option buffer_size '2097152'

```

保存并应用后，打开 http://路由器IP:8888/status 查看 udpxy 运行是否正常。当你在播放视频的时候，这个页面也会显示正在播放客户端的 IP 与它的实时流量。

![udpxy status](https://st.blackyau.net/blog/23/32.jpg)

然后你就可以在 PotPlayer 和 VLC media player 播放之前处理好的连接了，可以直接打开 M3U8 播放列表，也可以播放一个单独的地址。

例如你获取的地址为 `igmp://239.93.22.6:6666`

那么使用 udpxy 转换后的地址为 `http://192.168.10.1:8888/udp/239.93.22.6:6666`

如果你仍然无法播放，请将下面的防火墙规则添加进 `/etc/config/firewall`

如果你会使用 vim 那么直接在 Xshell 里面修改即可，如果你不会可以在 Xshell 窗口中点击  新建文本传输（Ctrl+Alt+F），将该文本下载到本地使用 notepad++ 进行修改，再上传上去。请注意你的防火墙配置可能已经存在，请你仔细的排查每一个设置项。

```
config rule
        option name 'Allow-IGMP'
        option target 'ACCEPT'
        option family 'ipv4'
        option src 'iptv'
        option proto 'IGMP'

config rule
        option name 'Allow-UDP-udpxy'
        option target 'ACCEPT'
        option src 'iptv'
        option proto 'udp'
        option dest_ip '224.0.0.0/4'

config rule
        option name 'Allow-UDP-igmpproxy'
        option target 'ACCEPT'
        option family 'ipv4'
        option src 'iptv'
        option proto 'udp'
        option dest 'lan'
        option dest_ip '224.0.0.0/4'
```

## 如何在外播放家中 IPTV 源

首先需要公网 IP，你可以在 在路由器 Web 端设置 - 网络 - 接口中，查看 WAN 获得的 IP 是否与你在 https://ip.sb/ 看到的 IP 一致。如果不一致的话，可以向电信人工客服反映「我需要公网 IP」即可。

首先你需要在 udpxy 配置中将 `Bind IP/Interface` 配置为 `0.0.0.0` 。

在路由器 Web 端设置 - 网络 - 防火墙 - 端口转发 中，添加协议为 tcp，外部区域为 wan，外部端口为 8888，内部 IP 地址为 192.168.10.1，内部端口为 8888 的规则即可。

![端口转发](https://st.blackyau.net/blog/23/33.jpg)

那么你在外要播放的话，只需要把路由器的 IP 地址换为你的公网 IP 即可。

例如你的本地播放地址为 `http://192.168.10.1:8888/udp/239.93.22.6:6666`

那么当你的公网 IP 为 `125.60.90.40` 时

你的互联网播放地址则为 `http://125.60.90.40:8888/udp/239.93.22.6:6666`

因为公网 IP 都在变，你可以使用 DDNS 也就是 动态 DNS 使用域名来访问，你可以使用路由器内自带的服务商。如果你和我一样将域名放置于 DNSPod 管理，也可以使用我制作的 [DdnsWithDnspod](https://github.com/blackyau/DdnsWithDnspod) 使用一个子域名来专供 IPTV 的播放。

## 结语

首先非常感谢各位前辈，我也是通过阅读现有的教程总结出来的。本文用了接近 5000 字，详细的介绍了有关 IPTV 与互联网的融合，希望能够对需要的朋友有帮助。因为本人能力有限，文中难免有一些问题也希望有发现的朋友能够及时的指出，我将感激不尽。

## 参考

[GitHub@coolsnowwolf - Lean's OpenWrt source](https://github.com/coolsnowwolf/lede)

[恩山无线论坛@鲲翔 - IPTV融合进普通网络一般步骤](https://www.right.com.cn/forum/thread-248400-1-1.html)

[恩山无线论坛@footlog - K2P/K2 padavan双线接入，宽带+IPTV，udpxy+xupnpd详细设置](https://www.right.com.cn/forum/thread-341748-1-1.html)

[恩山无线论坛@lcsuper - 小白的IPTV折腾教程(3)---双网融合、IPTV共享](https://www.right.com.cn/FORUM/thread-332086-1-1.html)

[恩山无线论坛@kangtao022 - 最新四川南充电信IPTV组播地址，及整理出地址列表的方法！](https://www.right.com.cn/forum/thread-508076-1-1.html)

[恩山无线论坛@橙子_MAX - 【附固件】全网首发，新三OpenWRT路由器IPTV内网融合视频教程](https://www.right.com.cn/forum/thread-596583-1-1.html)

[橙子的个人博客 - IPTV内网融合，实现任意设备观看IPTV](https://www.maxlicheng.com/openwrt/113.html)

[恩山无线论坛@angelkyo - 四川电信DHCP抓包能获取到IP，但是抓不到option60信息](https://www.right.com.cn/FORUM/thread-421870-1-1.html)

[恩山无线论坛@wengmingao - 简单的的IPTV 0成本抓包！](https://www.right.com.cn/FORUM/thread-329888-1-1.html)

[恩山无线论坛@莫问归期 - 在openwrt里安装udpxy后主题界面就会乱](https://www.right.com.cn/forum/thread-2304980-1-1.html)

[恩山无线论坛@xtwz - OpenWrt 编译 LuCI -> Applications 添加插件应用说明-L大](https://www.right.com.cn/forum/thread-344825-1-1.html)

[恩山无线论坛@happyzhang - OpenWrt入门编译 make menuconfig配置参考说明与自动生成脚本](https://www.right.com.cn/forum/thread-1237348-1-1.html)

[Github@MitchellJo - DdnsWithDnspod](https://github.com/MitchellJo/DdnsWithDnspod)

[恩山无线论坛@hayate - ppp拨号后自动运行脚本的问题](https://www.right.com.cn/forum/thread-52052-1-1.html)

[Demon's Blog@Demon - OpenWrt中的Hotplug脚本](http://demon.tw/hardware/openwrt-hotplug.html)

[痞客邦@Kai-Cho - [OpenWRT] hotplug](https://kevin0304.pixnet.net/blog/post/227990189)

[恩山无线论坛@ghostry - hotplug的iface脚本不执行怎么办?](https://www.right.com.cn/forum/thread-107357-1-1.html)

[OpenWrt Documentation - Hotplug](https://openwrt.org/docs/guide-user/base-system/hotplug)

[OpenWrt Documentation - udpxy](https://openwrt.org/docs/guide-user/services/proxy/udpxy)

[3mile博客 - 编译UDPXY新版本](https://3mile.github.io/archives/2019/1106111732/)

[Github@xingsiyue - udpxy无法保存缓存大小等参数](https://github.com/coolsnowwolf/lede/issues/1075)

[Github@jow- - luci-app-udpxy bug](https://github.com/openwrt/luci/issues/1494)