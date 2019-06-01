---
layout: post
title: Hadoop HA 搭建
date: 2019-05-28 15:54:12
updated: 2019-05-30 00:09:12
categories: 大数据
tags: 
    - CentOS
    - Linux
    - Hadoop
    - HDFS
    - YARN
    - Hadoop 集群管理
    - Hadoop HA
    - Hadoop 高可用
    - ZooKeeper
urlname: 16
comment: true
---

因为 HDFS 的 NameNode 存在单点问题，当它出现问题的时候整个 HDFS 都会无法访问。基于 ZooKeeper 搭建一个 Hadoop HA 高可用分布式署集群就尤为重要。

<!-- more -->

## 环境介绍

本次搭建的目标为，搭建 3 个 `DataNode` ，2个 `NameNode` ，2个 `yarn` 。并让两个 `NameNode` 做到能够异常自动切换， `yarn` 也同理。如下表：

| HostName | Function | IP |
| --- | --- | --- | --- |
| master | DataNode/NameNode/ResourceManager | 192.168.66.128 |
| slave1 | DataNode/NameNode/JobHistoryServer | 192.168.66.129 |
| slave2 | DataNode/ResourceManager | 192.168.66.130 |

软件版本如下：

| Program | Version | URL |
| --- | --- | --- |
| System | CentOS-7-x86_64-Minimal-1810 | [TUNA Mirrors](https://mirrors.tuna.tsinghua.edu.cn/centos/7.6.1810/isos/x86_64/) |
| JAVA | jdk-8u211-linux-x64.tar.gz | [Oracle](https://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html) |
| Hadoop | hadoop-2.6.0.tar.gz | [Apache Archive](http://archive.apache.org/dist/hadoop/common/hadoop-2.6.0/) |
| ZooKeeper | zookeeper-3.4.5.tar.gz | [Apache Archive](http://archive.apache.org/dist/zookeeper/zookeeper-3.4.5/) |

本文不会介绍理论性的东西，更多关于 `ZooKeeper` 和 `Hadoop HA` 定义相关的信息可以参考这个文章 [SegmentFault@Snailclimb - 可能是全网把 ZooKeeper 概念讲的最清楚的一篇文章](https://segmentfault.com/a/1190000016349824)

## 基础环境配置

参考 [Hadoop 伪分布部署](https://blackyau.cc/11.html) 和 [Hadoop 完全分布部署](https://blackyau.cc/14.html) 吧，这里不再多说。在开始配置之前吧所有相关服务都停止了再继续。

## ZooKeeper 配置

首先下载，解压 ZooKeeper

```shell
curl -O http://archive.apache.org/dist/zookeeper/zookeeper-3.4.5/zookeeper-3.4.5.tar.gz
tar xf zookeeper-3.4.5.tar.gz -C /usr/local/src/
vi ~/.bash_profile
```

在文本中添加以下内容

```
export ZOOKEEPER_HOME=/usr/local/src/zookeeper-3.4.5
PATH=$PATH:$ZOOKEEPER_HOME/bin
```

更新使其生效

```
source ~/.bash_profile
```

编辑 ZooKeeper 配置文件

```shell
cp /usr/local/src/zookeeper-3.4.5/conf/zoo_sample.cfg /usr/local/src/zookeeper-3.4.5/conf/zoo.cfg
vi /usr/local/src/zookeeper-3.4.5/conf/zoo.cfg
```

修改后如下

```cfg
# The number of milliseconds of each tick
tickTime=2000
# The number of ticks that the initial 
# synchronization phase can take
initLimit=10
# The number of ticks that can pass between 
# sending a request and getting an acknowledgement
syncLimit=5
# the directory where the snapshot is stored.
# do not use /tmp for storage, /tmp here is just 
# example sakes.
dataDir=/usr/local/src/zookeeper-3.4.5/data
dataLogDir=/usr/local/src/zookeeper-3.4.5/logs
# the port at which the clients will connect
clientPort=2181

server.1=master:2888:3888
server.2=slave1:2888:3888
server.3=slave2:2888:3888

#
# Be sure to read the maintenance section of the 
# administrator guide before turning on autopurge.
#
# http://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_maintenance
#
# The number of snapshots to retain in dataDir
#autopurge.snapRetainCount=3
# Purge task interval in hours
# Set to "0" to disable auto purge feature
#autopurge.purgeInterval=1
```

注意这里的 `dataDir` 不要放在 `/tmp` 或 `$HADOOP_HOME/tmp` 里面去，因为这两个目录都不能长久的保存数据，而 `ZooKeeper` 需要数据被长期保存。请注意，这里的配置需要在另外两台机子(slave1/slave2)上做同样的配置。可以直接使用 `scp` 传过去，然后手动配置 `~/.bash_profile` ，同时还需要手动创建一下文件夹。

```shell
# 在 master
scp -r /usr/local/src/zookeeper-3.4.5 slave1:/usr/local/src/
scp -r /usr/local/src/zookeeper-3.4.5 slave2:/usr/local/src/
```

```shell
# 每台机子都需要
mkdir /usr/local/src/zookeeper-3.4.5/logs
mkdir /usr/local/src/zookeeper-3.4.5/data
```

```shell
# 接下来在每台机子上都建立 myid 文件,并分别写入数字 1、2、3
[root@master ~]# echo 1 > /usr/local/src/zookeeper-3.4.5/data/myid
[root@slave1 ~]# echo 2 > /usr/local/src/zookeeper-3.4.5/data/myid
[root@slave2 ~]# echo 3 > /usr/local/src/zookeeper-3.4.5/data/myid
```

接下来每台机子上都启动一下同时查看运行是否正常。

```shell
zkServer.sh start
zkServer.sh status
```

如下有服务器进入了 `leader` 或 `follower` 模式即为启动成功

```shell
[root@master ~]# zkServer.sh status
JMX enabled by default
Using config: /usr/local/src/zookeeper-3.4.5/bin/../conf/zoo.cfg
Mode: follower

[root@slave1 ~]# zkServer.sh status
JMX enabled by default
Using config: /usr/local/src/zookeeper-3.4.5/bin/../conf/zoo.cfg
Mode: leader

[root@slave2 ~]# zkServer.sh status
JMX enabled by default
Using config: /usr/local/src/zookeeper-3.4.5/bin/../conf/zoo.cfg
Mode: follower
```

## Hadoop HA 配置

### core-site.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>

<!--指定nameservice的名称，自定义，但后面必须保持一致-->
<property>
 <name>fs.defaultFS</name>
 <value>hdfs://nscluster</value>
</property>

<property>
 <name>hadoop.tmp.dir</name>
 <value>/root/hadoop/tmp</value>
</property>

<!-- ZooKeeper服务器地址列表 -->
<property>
 <name>ha.zookeeper.quorum</name>
 <value>master:2181,slave1:2181,slave2:2181</value>
</property>

<!-- 主备NameNode切换时使用ssh登录上去杀掉进程 -->
<property>
 <name>dfs.ha.fencing.methods</name>
 <value>sshfence</value>
</property>

<!-- 指定ssh的密钥 -->
<property>
 <name>dfs.ha.fencing.ssh.private-key-files</name>
 <value>/root/.ssh/id_rsa</value>
</property>

</configuration>
```

### hdfs-site.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>

<property>
 <name>dfs.replication</name>
 <value>3</value>
</property>

<!--指定hdfs元数据存储的路径-->
<property>
    <name>dfs.namenode.name.dir</name>
    <value>file:/root/hadoop/tmp/data/nn</value>
</property>

<!--指定hdfs数据存储的路径-->
<property>
    <name>dfs.datanode.data.dir</name>
    <value>file:/root/hadoop/tmp/data/dn</value>
</property>

<!--关闭权限验证 -->
<property>
    <name>dfs.permissions.enabled</name>
    <value>false</value>
</property>

<!--以下为ha的相关配置-->
<!-- 指定hdfs的nameservice的名称为nscluster，务必与core-site.xml中的逻辑名称相同 -->
<property>
    <name>dfs.nameservices</name>
    <value>nscluster</value>
</property>    

<!-- 指定nscluster的两个namenode的名称，分别是nn1，nn2，注意后面的后缀.nscluster，这个是自定义的，如果逻辑名称为nsc，则后缀为.nsc，下面一样 -->
<property>
    <name>dfs.ha.namenodes.nscluster</name>
    <value>nn1,nn2</value>
</property>    

<!-- 配置nn1，nn2的rpc通信 端口    -->
<property>
    <name>dfs.namenode.rpc-address.nscluster.nn1</name>
    <value>master:9000</value>
</property>
<property>
    <name>dfs.namenode.rpc-address.nscluster.nn2</name>
    <value>slave1:9000</value>
</property>    

<!-- 配置nn1，nn2的http访问端口 -->
<property>
    <name>dfs.namenode.http-address.nscluster.nn1</name>
    <value>master:50070</value>
</property>
<property>
    <name>dfs.namenode.http-address.nscluster.nn2</name>
    <value>slave1:50070</value>
</property>    

<!-- 指定namenode的元数据存储在journalnode中的路径 -->
<property>
    <name>dfs.namenode.shared.edits.dir</name>
    <value>qjournal://master:8485;slave1:8485;slave2:8485/nscluster</value>
</property>    

<!-- 开启失败故障自动转移 -->
<property>
    <name>dfs.ha.automatic-failover.enabled</name>
    <value>true</value>
</property>     

<!-- 配置失败自动切换的方式 -->
<property>
    <name>dfs.client.failover.proxy.provider.nscluster</name>
    <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
</property>

</configuration>
```

### yarn-site.xml

```xml
<?xml version="1.0"?>

<configuration>

<property>
 <name>yarn.nodemanager.aux-services</name>
 <value>mapreduce_shuffle</value>
</property>

<!--以下为ha配置-->
<!-- 开启yarn ha -->
<property>
  <name>yarn.resourcemanager.ha.enabled</name>
  <value>true</value>
</property>

<!-- 指定yarn ha的名称 -->
<property>
  <name>yarn.resourcemanager.cluster-id</name>
  <value>nscluster-yarn</value>
</property>

<!--启用自动故障转移-->
<property>
    <name>yarn.resourcemanager.ha.automatic-failover.enabled</name>
    <value>true</value>
</property>

<!-- resourcemanager的两个名称 -->
<property>
  <name>yarn.resourcemanager.ha.rm-ids</name>
  <value>rm1,rm2</value>
</property>

<!-- 配置rm1、rm2的主机  -->
<property>
  <name>yarn.resourcemanager.hostname.rm1</name>
  <value>master</value>
</property>
<property>
  <name>yarn.resourcemanager.hostname.rm2</name>
  <value>slave2</value>
</property>

<!-- 配置yarn web访问的端口 -->
<property>
  <name>yarn.resourcemanager.webapp.address.rm1</name>
  <value>master:8088</value>
</property>
<property>
  <name>yarn.resourcemanager.webapp.address.rm2</name>
  <value>slave2:8088</value>
</property>

<!-- 配置zookeeper的地址 -->
<property>
  <name>yarn.resourcemanager.zk-address</name>
  <value>master:2181,slave1:2181,slave2:2181</value>
</property>

<!-- 配置zookeeper的存储位置 -->
<property>
  <name>yarn.resourcemanager.zk-state-store.parent-path</name>
  <value>/rmstore</value>
</property>

<!--  yarn restart-->
<!-- 开启resourcemanager restart -->
<property>
  <name>yarn.resourcemanager.recovery.enabled</name>
  <value>true</value>
</property>

<!-- 配置resourcemanager的状态存储到zookeeper中 -->
<property>
  <name>yarn.resourcemanager.store.class</name>
  <value>org.apache.hadoop.yarn.server.resourcemanager.recovery.ZKRMStateStore</value>
</property>

<!-- 开启nodemanager restart -->
<property>
  <name>yarn.nodemanager.recovery.enabled</name>
  <value>true</value>
</property>

<!-- 配置rpc的通信端口 -->
<property>
  <name>yarn.nodemanager.address</name>
  <value>0.0.0.0:45454</value>
</property>

</configuration>
```

### mapred-site.xml

```xml
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>

<configuration>

<property>
 <name>mapreduce.framework.name</name>
 <value>yarn</value>
</property>

</configuration>
```

将配置文件同步到所有主机

```shell
scp -r /usr/local/hadoop-2.6.0/etc/hadoop slave1:/usr/local/hadoop-2.6.0/etc/
scp -r /usr/local/hadoop-2.6.0/etc/hadoop slave2:/usr/local/hadoop-2.6.0/etc/
```

## 启动

```shell
# 每台机子都要执行一次
zkServer.sh start
```

```shell
# master 
hadoop-daemons.sh start journalnode # 所有主机启动journalnode集群(带s可以一条命令启动集群)
hdfs zkfc -formatZK # 格式化zkfc
hadoop namenode -format # 格式化hdfs
hadoop-daemon.sh start namenode # 本机启动NameNode
hadoop-daemons.sh start datanode # 所有主机启动DataNode
start-yarn.sh # 本机启动yarn
```

```shell
# slave1
hdfs namenode -bootstrapStandby # 启动数据同步
hadoop-daemon.sh start namenode # 本机启动NameNode
mr-jobhistory-daemon.sh start historyserver # 启动历史服务器
```

```shell
# slave2
yarn-daemon.sh start resourcemanager # 启动yarn备用节点
```

```shell
# master
hadoop-daemons.sh start zkfc # 开启zkfc
```

最后一步完成时，两个 `NameNode` 的其中一个就会变为 `active`

## 测试

http://master:50070

http://master:8088

http://slave1:50070

http://slave1:19888

http://slave2:8088

```shell
[root@master ~]# jps
10003 DataNode
10852 QuorumPeerMain
10948 DFSZKFailoverController
9797 JournalNode
13050 Jps
13004 ResourceManager
9870 NameNode
11150 NodeManager
```

```shell
[root@slave1 ~]# jps
7379 DataNode
7301 JournalNode
8070 NodeManager
7975 DFSZKFailoverController
8218 JobHistoryServer
8778 Jps
7902 QuorumPeerMain
7615 NameNode
```

```shell
[root@slave2 ~]# jps
7317 JournalNode
7765 QuorumPeerMain
7989 ResourceManager
7880 NodeManager
7385 DataNode
9839 Jps
```

## 参考

[Hadoop: The Definitive Guide@Tom White](https://item.jd.com/12109713.html)

[51CTO博客@maisr25 - hadoop2.0 HA的主备自动切换](https://blog.51cto.com/sstudent/1388865)

[51CTO博客@maisr25 - hadoop2.0 QJM方式的HA的配置](https://blog.51cto.com/sstudent/1381674)

[博客园@learn21cn - zookeeper集群的搭建以及hadoop ha的相关配置](https://www.cnblogs.com/learn21cn/p/6184490.html)

[博客园@黄石公园 - 大数据系列（hadoop） Hadoop+Zookeeper 3节点高可用集群搭建](https://www.cnblogs.com/YellowstonePark/p/7750213.html)

[博客园@一蓑烟雨任平生 - hadoop集群搭建（伪分布式）+使用自带jar包计算pi圆周率](https://www.cnblogs.com/jingpeng77/p/9652380.html)