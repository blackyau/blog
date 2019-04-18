---
layout: post
title: Hadoop 伪分布部署
date: 2019-04-04 17:00:17
updated: 2019-04-09 13:31:22
categories: 大数据
tags: 
    - CentOS
    - Linux
    - Hadoop
    - HDFS
    - YARN
    - MapReduce
    - VMware
urlname: 11
comment: true
---

单机安装 Hadoop 还是比较简单，这里就使用 VMware 模拟部署一下。年轻人的一次大数据之旅？？

<!-- more -->

## 新建虚拟机

下载镜像：[https://mirrors.tuna.tsinghua.edu.cn/centos/7/isos/x86_64/CentOS-7-x86_64-Minimal-1810.iso](https://mirrors.tuna.tsinghua.edu.cn/centos/7/isos/x86_64/CentOS-7-x86_64-Minimal-1810.iso)

![1](https://st.blackyau.net/blog/11/1.png)
![2](https://st.blackyau.net/blog/11/2.png)
![3](https://st.blackyau.net/blog/11/3.png)
![4](https://st.blackyau.net/blog/11/4.png)
![5](https://st.blackyau.net/blog/11/5.png)

内存和硬盘大小请根据情况而定。其次，我一般习惯在创建完虚拟机后，进入虚拟机设置将打印机移除再启动。

按方向键将焦点移动至上方 `Install CentOS 7` 并回车

![6](https://st.blackyau.net/blog/11/6.png)

直接点 `Continue` 使用英语的话遇到问题方便在网上找解决方法

![7](https://st.blackyau.net/blog/11/7.png)

首先为了查 `Log` 方便，进入 `DATE & TIME` 把时区了时间调到和本机一样。

点进 `NETWORK & HOST NAME` 把网络连接打开

最后点击开始安装

![8](https://st.blackyau.net/blog/11/8.png)

安装的时候可以设置 `Root` 密码，安装完了直接点 `Reboot` 重启就完事儿了

![9](https://st.blackyau.net/blog/11/9.png)

等待它出现

```shell
CentOS Linux 7 (Core)
Kernel 3.10.0-957.e17.x86_64 on an x86_64

localhost login:
```

的时候就说明安装完毕了，然后我们输入 `root` 和你刚刚设置的密码。登陆系统后输入 `ip add` 查询一下本机IP。

![10](https://st.blackyau.net/blog/11/10.png)

有了本机 `IP` 就可以换用 Xshell, putty, secureCRT 之类的软件使用 SSH 连接了。可以快乐的复制粘贴了，这里我以 Xshell 为例。

> Xshell 的 复制 快捷键为 `Ctrl+Insert` , 粘贴 快捷键为 `Shift+Insert` (在标准键盘中 Insert 在 Delete 的上面)

新建会话填写 `名称` 和 `IP` ，然后点击侧栏的 `用户身份验证` 填入 `用户名` 和 `密码`

![11](https://st.blackyau.net/blog/11/11.png)

再然后点击侧栏 `SSH` 里面的 `隧道` ，将下面的 `转发X11连接到` 关闭。

![12](https://st.blackyau.net/blog/11/12.png)

点击确定，连接后提示 `未知主机密钥` 选择 `接收并保存` 即可

![13](https://st.blackyau.net/blog/11/13.png)

## 服务器基本环境配置

### 关闭防火墙

```shell
systemctl stop firewall # 关闭防火墙
systemctl disable firewall # 关闭防火墙自启
firewall-cmd --state # 检查防火墙状态
```

当它输出以下提示时意味着这一步你已经完成了

```shell
not running
```

### 关闭 Selinux

> 想了解有关它的更多信息请自行搜索

首先打开配置文件

```shell
vi /etc/selinux/config
```

修改字段如下

```shell
# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
SELINUX=disabled # 将这里改为 disabled
# SELINUXTYPE= can take one of three values:
#     targeted - Targeted processes are protected,
#     minimum - Modification of targeted policy. Only selected processes are protected.
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted
```

> 保存并退出 vi 的方法是 按 esc 后输入 :wq 。有关 vi 的更多操作请自行搜索

随后输入以下指令重启以生效

### 修改 hostname

```shell
hostnamectl set-hostname master # 修改 hostsname 为 master
vi /etc/hosts # 打开 hosts 文件
```

由于 `HDFS` 钟爱于使用 `localhosts` 所以要将 `hosts` 里面有关的信息都注释掉，修改后 `hosts` 文件如下

```conf
# 127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
# ::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.66.135 master # 这里应该改成你自己的 IP
```

```shell
reboot # 重启生效
```

## 安装 Java

下载 [https://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html](https://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)

我选择了 `Linux x64 rpm 安装包`

在下载目录的空白处按住 `Shift` 点击鼠标右键，然后点击 `在此处打开 PowerShell 窗口` 召唤出来 `PowerShell` ，因为本机有 Python 环境，所以就使用 `python -m http.server 80` 启动一个简易的 `http服务` 。然后用浏览器打开 `服务器IP的默认网关` 一般情况下与 服务器IP 的差别只有 `最后一位为1` 。这里我就选择打开 `192.168.66.1`。并复制 `Java JDK 链接`。

> 别忘了关闭 Windows 防火墙，python2(CentOS7 自带) 的http服务命令是 `python -m SimpleHTTPServer 80`

![14](https://st.blackyau.net/blog/11/14.png)

在 CenOS 下输入以下命令下载并安装

```shell
curl -O http://192.168.66.1/jdk-8u201-linux-x64.rpm #这里应该替换为你刚刚复制的链接
rpm -ivh jdk-8u201-linux-x64.rpm
```

当你看到控制台输出类似信息说明你以安装成功

```shell
[root@master ~] curl -O http://192.168.66.1/jdk-8u201-linux-x64.rpm
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  168M  100  168M    0     0  7569k      0  0:00:22  0:00:22 --:--:-- 7910k
[root@master ~] rpm -ivh jdk-8u201-linux-x64.rpm
warning: jdk-8u201-linux-x64.rpm: Header V3 RSA/SHA256 Signature, key ID ec551f03: NOKEY
Preparing...                          ################################# [100%]
Updating / installing...
   1:jdk1.8-2000:1.8.0_201-fcs        ################################# [100%]
Unpacking JAR files...
	tools.jar...
	plugin.jar...
	javaws.jar...
	deploy.jar...
	rt.jar...
	jsse.jar...
	charsets.jar...
	localedata.jar...
```

测试一下把

```shell
java -version
```

输出以下信息说明安装成功

```shell
java version "1.8.0_201"
Java(TM) SE Runtime Environment (build 1.8.0_201-b09)
Java HotSpot(TM) 64-Bit Server VM (build 25.201-b09, mixed mode)
```

## 安装 Hadoop

下载 [https://archive.apache.org/dist/hadoop/core/hadoop-2.6.0/hadoop-2.6.0.tar.gz](https://archive.apache.org/dist/hadoop/core/hadoop-2.6.0/hadoop-2.6.0.tar.gz)

还是使用老办法将文件传进 CentOS

```shell
curl -O http://192.168.66.1/hadoop-2.6.0.tar.gz # 这里应该改成你复制的链接
tar -xvf hadoop-2.6.0.tar.gz # 解压 Hadoop
```

接下来我们要为 JAVA 和 Hadoop 配置环境变量，首先打开配置文件 `vi /etc/profile` 按一下大写 `G` 转跳到文本底部，然后添加内容后文本底部如下。

```shell
for i in /etc/profile.d/*.sh /etc/profile.d/sh.local ; do
    if [ -r "$i" ]; then
        if [ "${-#*i}" != "$-" ]; then
            . "$i"
        else
            . "$i" >/dev/null
        fi
    fi
done

unset i
unset -f pathmunge

export JAVA_HOME=/usr/java/jdk1.8.0_201-amd64
export HADOOP_HOME=/root/hadoop-2.6.0
export PATH=$PATH:$JAVA_HOME/bin:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
```

输入以下内容让系统更新配置文件

```shell
source /etc/profile
```

输入以下内容检测配置是否正常

```shell
hadoop version
```

当返回如下信息时说明你的配置已经成功了

```shell
Hadoop 2.6.0
Subversion https://git-wip-us.apache.org/repos/asf/hadoop.git -r e3496499ecb8d220fba99dc5ed4c99c8f9e33bb1
Compiled by jenkins on 2014-11-13T21:10Z
Compiled with protoc 2.5.0
From source with checksum 18e43357c8f927c0695f1e9522859d6a
This command was run using /root/hadoop-2.6.0/share/hadoop/common/hadoop-common-2.6.0.jar
```

## 配置 SSH 实现免密登陆

```shell
ssh-keygen -t rsa 
```

然后一直回车，输出信息如下即可

```shell
[root@master hadoop-2.6.0] ssh-keygen -t rsa 
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa): 
Created directory '/root/.ssh'.
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:bP4nV0fky1ShxRm07JRVIYIrpdb23O1ha+xIJYONTTY root@master
The key's randomart image is:
+---[RSA 2048]----+
|   .      .. .oL+|
|     B   o  . *o*|
|    B o + .  . Bo|
|   o B @ o    ..=|
|    . @ P     oo.|
|     . O      .o.|
|    . . .    . . |
|     . . .. o    |
|          .+     |
+----[SHA256]-----+
```

生成 authorized_keys

```shell
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
```

使用以下命令测试能否实现免密码登陆

```shell
ssh master
```

如果你只输入了 `Yes` 就登陆成功了，说明这一步你配置成功了。然后输入 `exit` 退出后进行下一步配置。

## 配置 Hadoop

> 开始配置之前我建议你创建一次快照

进入 Hadoop 目录

```shell
cd /root/hadoop-2.6.0/etc/hadoop/
```

### core-site.xml

```shell
vi core-site.xml
```

在 `<configuration>` 和 `</configuration>` 之间添加以下内容

```xml
<!-- 指定 HADOOP 所使用的文件系统 schema（URI），HDFS 的老大（NameNode）的地址 -->
<property>
    <name>fs.defaultFS</name>
    <value>hdfs://master:9000</value> # 本机 hostname
</property>

<!-- 指定 hadoop 运行时产生文件的存储目录 -->
<property>
    <name>hadoop.tmp.dir</name>
    <value>/root/hadoop-2.6.0/tmp</value> # 储存目录
</property>
```

### hdfs-site.xml 

`vi hdfs-site.xml`

在 `<configuration>` 和 `</configuration>` 之间添加以下内容

```xml
<!-- 指定 HDFS 副本的数量 -->
<property>
    <name>dfs.replication</name>
    <value>1</value> # 伪分布,数据只储存在1个地方
</property>
```

### mapred-site.xml

```shell
cp mapred-site.xml.template mapred-site.xml
vi mapred-site.xml
```

在 `<configuration>` 和 `</configuration>` 之间添加以下内容

```xml
<!-- 指定 mr 运行在 yarn 上 -->
<property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
</property>
```

### yarn-site.xml

`vi yarn-site.xml`

```xml
<!-- 指定 YARN 的老大（ResourceManager）的地址 -->
<property>
    <name>yarn.resourcemanager.hostname</name>
    <value>master</value>
</property>

<!-- reducer 获取数据的方式 -->
<property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
</property>
```

### hadoop-env.sh

```shell
vi hadoop-env.sh
```

在文档最下面新增一行

```xml
export JAVA_HOME=/usr/java/jdk1.8.0_201-amd64
```

## 运行 Hadoop

```shell
hdfs namenode -format # 格式化 HDFS
start-dfs.sh # 启动 HDFS
start-yarn.sh # 启动 YARN
mr-jobhistory-daemon.sh start historyserver # 启动 MapReduce
hdfs dfs -chmod -R 755 /tmp # 赋予目录权限(这个问题暂时先这样把,最好还是用两个用户把它分离开.不直接使用root用户)
start-all.sh # 很有把握的时候才用这个
```

## 停止 Hadoop

调整了配置后，一定要先停止再启动

```shell
stop-dfs.sh # 停止 HDFS
stop-yarn.sh # 停止 YARN
mr-jobhistory-daemon.sh stop historyserver # 停止 MapReduce
stop-all.sh # 停止所有
```

## 测试

http://192.168.66.135:50070 # NameNode

![15](https://st.blackyau.net/blog/11/15.png)

http://192.168.66.135:19888 # 资源管理器

![16](https://st.blackyau.net/blog/11/16.png)

http://192.168.66.135:8088 # 历史服务器

![17](https://st.blackyau.net/blog/11/17.png)

## 参考

[Hadoop: The Definitive Guide@Tom White](https://item.jd.com/12109713.html)

[csdn@dancheng_work - centos 7关闭防火墙](https://blog.csdn.net/dancheng1/article/details/78512028)

[博客园@WhoAmMe - CentOS添加环境变量](https://www.cnblogs.com/whoamme/p/4039998.html)