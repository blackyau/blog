---
layout: post
title: Hadoop 完全分布部署
date: 2019-04-24 20:00:29
updated: 2019-04-25 16:18:40
categories: 大数据
tags: 
    - CentOS
    - Linux
    - Hadoop
    - HDFS
    - YARN
    - MapReduce
    - VMware
urlname: 14
comment: true
---

之前有说过Hadoop伪分布的部署，这次来讲讲完全分布。总体来说和伪分布的配置差别不大，只是不同机子之前的衔接和小部分的配置修改。

<!-- more -->

这次就不像以前的傻瓜式教程了，这次主要说说在这一次搭建中遇到的一些问题，同时也是对之前伪分布搭建的补充。

## 安装 CentOS

之前分配只分配了 `1G` 内存太小了，至少要给 `master` 分配 `4G` 的内存。在安装的时候可以直接打开网络，同时吧 `hostname` 设置好，省的后面又要自己去改。先打开网络之后，就用打开一下 `网络时间同步` 同时吧时区调到和真机一样，方便调试。磁盘分区什么的直接自动走起就行了。

## 基础环境配置

关 `selinux` 和关 `防火墙` 是必须的，省了很多麻烦。然后我这次在部署的时候，为每个程序都分了一个用户(hadoop, yarn, hdfs, mapred 都在 `hadoop group` 里面)。这就和之前无脑使用 `root` 有点不一样，要很注意权限的问题。冷不丁顶的就会 `Permission denied` ，而且文件被新建的权限同组用户是只有读权限的。每次都要手动新建一下，然后 `chmod g+w xxxxxx` 挺讲究。其次就是 `ssh` 的配置，因为 `hdfs` 和 `yarn` 在不同的机子之间有交互，所以要把所有的 `id_rsa.pub` 都放在同一个 `authorized_key` 里面，每个机子的 `hdfs` 和 `yarn` 用户都要，然后每一行一个，如下：

```shell
ssh-rsa AAAAB3N......KcSd+lF9EYT7ED9KMWlajl yarn@master
ssh-rsa AAAAB3N......pggVZIG3+ElmBlPqYp3O8D hdfs@master
ssh-rsa AAAAB3N......a2X44NMnhAbcQVDcMwdoIA hdfs@slave1
ssh-rsa AAAAB3N......PCBqtandHgi0CwKsLsCv2d yarn@slave1
ssh-rsa AAAAB3N......GXNfdgrthrtgf0+1DrYHGp hdfs@slave2
ssh-rsa AAAAB3N......PXvHKFTQ2b8Xt8ZAvB/dKy yarn@slave2
```

## Hadoop配置

配置上面其实和伪分布差不了太多，只有一些小地方有变化。而且很方便的就是，所有机子的配置文件是一样的。所以不需要对每个单独的写配置，无脑的复制粘贴就好了。在机子比较多的情况下，还有很多可以自动化同步配置的工具，我这里目前用不上也没了解，就不多说了。直接看看配置吧。

### core-site.xml

```xml
<configuration>
<!-- Hadoop Core 配置项，用于设置 HDFS、MapReduce和YARN常用的 I/O 设置 -->

<property>
<!-- 默认文件系统，端口默认值为 8020 -->
  <name>fs.defaultFS</name>
  <value>hdfs://master/</value>
</property>

<property>
<!-- HDFS的储存目录,HDSF里面的数据都该目录内 -->
  <name>hadoop.tmp.dir</name>
  <value>/usr/local/hadoop-2.6.0/tmp</value>
</property>

</configuration>
```

### hdfs-site.xml

```xml
<configuration>
<!-- HDFS的配置,因为我没什么需求挺多目录都跟随上面的 hadoop.tmp.dir 走就行了-->

<property>
 <!-- HDFS的默认副本数，其实这里的值默认就是3可以不用改 -->
    <name>dfs.replication</name>
    <value>3</value>
</property>

</configuration>
```

### yarn-site.xml

```xml
<configuration>

<!-- YARN 守护进程的配置项，包括资源管理器，WEB应用代理服务器和节点管理器 -->

<property>
<!-- 指定一台电脑作为资源管理器 -->
  <name>yarn.resourcemanager.hostname</name>
  <value>master</value>
</property>

<property>
<!-- 看官方文档说这里的默认是就是这个，但是书上说需要手动设置才能运行正常。后面有时间再深究 -->
  <name>yarn.nodemanager.aux-services</name>
  <value>mapreduce_shuffle</value>
</property>

</configuration>
```

### mapred-site.xml

```xml
<configuration>
<!-- 这个文件本身是不存在的，需要手动 cp 一下改个名字。字如其名，MapReduce的配置文件 -->
<property>
<!-- 指定 MapReduce 的使用框架，这里如果不指定的话他就默认使用的 local 模式，而且 8088 里面看不到 job-->
  <name>mapreduce.framework.name</name>
  <value>yarn</value>
</property>

</configuration>
```

### slaves

这是一个和其他配置文件同一目录的纯文本，它里面定义了该集群中所有主机的 `hostname` 或 `ip` ，这里我使用的 `hostname`

```
master
slave1
slave2
```

## 运行

还需要注意的就是运行的时候，还是因为我使用了不同的用户管理不同的进程。所以运行的时候要来回切换用户，下面是不同服务启动时所对应的用户。

| User | Start command |
| ---- | ----- |
| `hdfs` | `hdfs namenode -format` |
| `hdfs` | `start-dfs.sh` |
| `yarn` | `start-yarn.sh` |
| `mapred` | `mr-jobhistory-daemon.sh start historyserver` |

JobHistory WEB 端能够正常显示，但是它不显示 `job` 。查 `log` 发现还是权限的问题，需要给一下权限。`hdfs dfs -chown -R mapred /tmp/hadoop-yarn/staging/history`上面的不见效就直接 777 `hdfs dfs -chmod 777 /tmp/hadoop-yarn/staging/history` 。

如果启动的时候失败了，数据没有成功同步到所有节点。就吧 `hadoop.tmp.dir` 和 `/tmp` 目录里面的东西全部删完，重新格式化就行了。而且注意不要格式化太多次，格式化太多次导致 `ID` 不一样也会是一个很头大的事情，[之前](https://blackyau.cc/12.html#%E6%8E%92%E9%94%99)因为这个问题我头疼了 1-2 天。

## HIVE

HIVE 启动的时候，还是要注意删一下和 `Hadoop` 冲突的 `jar` ,其中任选一个删除都可以

```shell
/usr/local/hadoop-2.6.0/share/hadoop/common/lib/slf4j-log4j12-1.7.5.jar
/usr/local/hadoop-2.6.0/share/hadoop/yarn/lib/hive-jdbc-1.1.0-standalone.jar
```

也可以用环境变量，让他自己选择高版本的使用。但是这样设置的话，还是会有一串警告，看着挺不爽的。

```shell
export HADOOP_USER_CLASSPATH_FIRST=true
```

## 参考

[Hadoop: The Definitive Guide@Tom White](https://item.jd.com/12109713.html)