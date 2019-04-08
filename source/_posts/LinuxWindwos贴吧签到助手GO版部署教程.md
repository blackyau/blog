---
layout: post
title: Linux/Windwos贴吧签到助手GO版部署教程
date: 2016-10-30 17:38:00
updated: 2016-10-31 15:26:00
categories: 教程
tags: 
    - Linux
    - CentOS
    - 贴吧签到
    - Windows
    - GO
    - kookxiang
urlname: 7
comment: true
---
前阵子和几位小伙伴一起合租的一个VPS用来扶墙，只扶墙有点浪费的感觉。前阵子部署博客的服务商把我无名智者的贴吧签到给端了，正好趁这次机会就把kookxiang的GO版签到用上了！顺便也就写了这一篇文章，希望大家在部署的时候能够少走一些弯路。

<!-- more -->

本人的部署环境：windows7 SP1 / CentOS 6.8

# windows端配置

首先到[贴吧签到助手GO版GitHub页](https://github.com/kookxiang/Tieba_Sign-Go/releases)下载最新版主程序，你可以通过右键 计算机-属性 来看到自己的系统是64位还是32位。Tieba_Sign-Go.x64.exe对应64位系统，Tieba_Sign-Go.x86.exe对应32位。

接下来就是获取Cookie的时候，首先在浏览器中使用隐身模式（为了防止退出账号后Cookie失效），打开[百度贴吧](https://tieba.baidu.com)并登录你自己的帐号。然后按一下F12打开 开发者工具，并切换到Network。在不关闭开发者工具的情况下，刷新一下页面。
![img](https://st.blackyau.net/blog/7/1.png)

单击第一条请求并切换到Headers
![img](https://st.blackyau.net/blog/7/2.png)

将Cookie项目中的数值复制出来，放进一个任意名字的TXT文档。（不带"Cookie:"）
将文本中所有的"冒号+空格"全部使用回车替换（嫌麻烦可以用Word进行替换）
![img](https://st.blackyau.net/blog/7/3.png)

同时在贴吧签到助手GO版主程序的同一目录中，新建一个名为cookies的文件夹。并将你刚刚，储存了cookie数值的TXT文档放置其中。
![img](https://st.blackyau.net/blog/7/4.png)

右键主程序 发送到 桌面快捷方式，再右键桌面的快捷方式进入属性。将目标改为

    "主程序路径" -batch
    例如↓
    "C:\Tieba_Sign-Go\Tieba_Sign-Go.x86.exe" -batch

现在你就可以直接双击打开，完成签到。你还可以将本快捷方式，属性中的运行方式调整为最小化运行。并放置于开始的“启动”中。这样它就可以在每天开机的时候为你默默的签到，你也可以同时在使用 任务计划程序 在每天的指定时间为你补签。具体的设置方法我这里就不做过多的介绍了，值得一提的就是在设置 操作 的时候运行的程序要选主程序。要添加参数-batch，不能够直接运行快捷方式。

按同样的方法，你还可以在Cookies目录中添加更多帐号。同时完成更多的签到任务体验GO签到带来的迅猛速度。

# Linux端配置
获取Cookie的方法我就不在赘述，直接就说说部署的方法了。

首先使用[WinSCP](http://winscp.net/eng/docs/lang:chs)将对应版本的GO签到的主程序(一般都是Tieba_Sign-Go.linux.386.bin)放置于/ROOT/tb同时和windows端的操作一样，将你获取到的Cookies都放置于cookies文件夹中。

    cd /ROOT/tb  //切换到/ROOT/tb目录
    vi run.sh    //编辑一个脚本，用于带参数运行主程序

在run.sh中写入以下内容

    #!/bin/bash
    /root/tb/Tieba_Sign-Go.linux.386.bin -batch

并赋予其运行权限

    chmod +x run.sh
    chmod +x *.bin

手动运行一下脚本看看能否正常签到

    ./run.sh

安装/配置crontab用于定时运行签到程序（以下所有指令都不要输入//及以后的文字，那是用于解释该行指令的用途）
首先执行以下指令启动crontab

    service crond start
    若返回的提示没有成功启动等字样你就需要手动安装crontab

CentOS的安装方法如下

    yum install vixie-cron  //途中会问题Y/N你输入Y按回车就行
    yum install crontabs  //同上

Ubuntu的安装方法如下

    apt-get install cron

安装完毕后再执行`service crond start`启动服务就行

    crontab -e  //本用于编辑任务，此教程用于生成配置文件
    vi /etc/crontab  //编辑任务配置

在配置中最下面一行写入，用于每天2:01的时候签到。按Esc退出，输入:wq保存

    1  2  *  *  * root batch /root/tb/Tieba_Sign-Go.linux.386.bin

这里值得一提的是，你要看看自己服务器是用的什么时区的时间。比如我的日本VPS就比北京时间快了1小时。

    date -R  //查看VPS当前时间，别忘了R要大写
    service crond reload  //重新载入配置
    chkconfig --level 35 crond on  //加入开机启动项

就这样，你的VPS就会在你指定的时间，自动签到啦。关于时间的设置，其实还有很多可变性。你可以通过Google或百度crontab找到很多例子，我这里就不说其他的了。提前祝大家万圣节快乐！