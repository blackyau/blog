---
layout: post
title: HBase 配置
date: 2019-06-01 18:09:22
updated: 2019-06-02 17:08:10
categories: 大数据
tags: 
    - CentOS
    - Linux
    - Hadoop
    - HDFS
    - HBase
    - Hadoop HA
    - Hadoop 高可用
    - ZooKeeper
urlname: 17
comment: true
---

我接触的第一个 非关系型数据库 就是 HBase ,有关它的更多概念我这里就不说了。本文关注的是它的搭建、配置、使用。

<!-- more -->

## 环境介绍

软件版本如下：

| Program | Version | URL |
| --- | --- | --- |
| System | CentOS-7-x86_64-Minimal-1810 | [TUNA Mirrors](https://mirrors.tuna.tsinghua.edu.cn/centos/7.6.1810/isos/x86_64/) |
| JAVA | jdk-8u211-linux-x64.tar.gz | [Oracle](https://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html) |
| Hadoop | hadoop-2.6.0.tar.gz | [Apache Archive](http://archive.apache.org/dist/hadoop/common/hadoop-2.6.0/) |
| ZooKeeper | zookeeper-3.4.5.tar.gz | [Apache Archive](http://archive.apache.org/dist/zookeeper/zookeeper-3.4.5/) |
| HBase | hbase-1.2.0-bin.tar.gz | [Apache Archive](http://archive.apache.org/dist/hbase/1.2.0/) |

关于版本的问题，我这里使用的环境并不是官方推荐的组合。在官方文档上面有关 Hadoop 不同版本与 HBase 的兼容性有介绍，可以看[这里](https://hbase.apache.org/book.html#hadoop)

> 在 Hadoop 2.6.x 中的 Hadoop 2.6.0 版本下运行 HBase 可能会导致集群故障和数据丢失，请使用 Hadoop 2.6.1+ 版本

## 目标

- 完成 HBase 单机模式配置
- 完成 HBase 分布模式配置
- HBase 数据的导出/导入

## 基础环境配置

参考 [Hadoop HA 搭建](https://blackyau.cc/16.html) 目前已完成 Hadoop HA 环境搭建

| HostName | Function | IP |
| --- | --- | --- | --- |
| master | DataNode/NameNode/ResourceManager | 192.168.66.128 |
| slave1 | DataNode/NameNode/JobHistoryServer | 192.168.66.129 |
| slave2 | DataNode/ResourceManager | 192.168.66.130 |

### 下载解压

首先下载，解压 HBase

```shell
curl -O http://archive.apache.org/dist/hbase/1.2.0/hbase-1.2.0-bin.tar.gz
tar xf hbase-1.2.0-bin.tar.gz -C /usr/local/src/
```

### 系统环境变量

配置 HBase 环境变量，只对当前用户生效

```shell
vi ~/.bash_profile
```

添加以下内容

```shell
export HBASE_HOME=/usr/local/src/hbase-1.2.0
PATH=$PATH:$HBASE_HOME/bin
```

使其生效

```shell
source ~/.bash_profile
```

测试是否配置成功

```shell
hbase version
```

输出以下信息说明配置成功

```shell
HBase 1.2.0
Source code repository git://asf-dev/home/busbey/projects/hbase revision=25b281972df2f5b15c426c8963cbf77dd853a5ad
Compiled by busbey on Thu Feb 18 23:01:49 CST 2016
From source with checksum bcb25b7506ecf5d62c79d8f7193c829b
```

### hbase-env

把 JAVA_HOME 写进 HBase 环境变量

```shell
vi /usr/local/src/hbase-1.2.0/conf/hbase-env.sh
```

添加以下内容

```shell
export /usr/local/src/jdk1.8.0_211/
```

## 单机模式

### hbase-site.xml单机模式

打开主配置文件

```shell
vi /usr/local/src/hbase-1.2.0/conf/hbase-site.xml
```

将光标放在第一行，输入以下命令清空配置文件

```shell
:.,$d
```

写入以下内容,配置来自 [HBase Doc v1.2](https://hbase.apache.org/1.2/book.html#_get_started_with_hbase)

```xml
<configuration>
  <property>
    <name>hbase.rootdir</name>
    <value>file:///root/standalone/hbase</value>
    <!-- 设置储存目录 -->
  </property>
  <property>
    <name>hbase.zookeeper.property.dataDir</name>
    <value>/root/standalone/zookeeper</value>
    <!-- ZooKeeper目录 -->
  </property>
</configuration>
```

### 运行单机模式

```shell
start-hbase.sh
```

查看进程是否在运行

```shell
[root@master ~]# jps
10082 HMaster
10346 Jps
```

到这里单机模式就配置完成了

## 分布式模式

### hbase-site.xml分布式模式

```shell
vi /usr/local/src/hbase-1.2.0/conf/hbase-site.xml
```

将光标放在第一行，输入以下命令清空配置文件

```shell
:.,$d
```

写入以下内容,配置来自 [HBase Doc v1.2](https://hbase.apache.org/1.2/book.html#example_config)

```xml
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
  <property>
    <name>hbase.zookeeper.quorum</name>
    <value>master,slave1,slave2</value>
    <!-- 集群主机的 hostname -->
    <description>The directory shared by RegionServers.
    </description>
  </property>
  <property>
    <name>hbase.zookeeper.property.dataDir</name>
    <value>/export/zookeeper</value>
    <description>Property from ZooKeeper config zoo.cfg.
    The directory where the snapshot is stored.
    </description>
  </property>
  <property>
    <name>hbase.rootdir</name>
    <value>hdfs://nscluster:8020/hbase</value>
    <!-- 注意这里要和你 Hadoop hdfs-site.xml 配置中的 fs.defaultFS 设置相同 -->
    <description>The directory shared by RegionServers.
    </description>
  </property>
  <property>
    <name>hbase.cluster.distributed</name>
    <value>true</value>
    <description>The mode the cluster will be in. Possible values are
      false: standalone and pseudo-distributed setups with managed Zookeeper
      true: fully-distributed with unmanaged Zookeeper Quorum (see hbase-env.sh)
    </description>
  </property>
</configuration>
```

### regionservers

修改集群节点信息配置文件

```shell
vi /usr/local/src/hbase-1.2.0/conf/regionservers
```

清空所有内容，写入以下内容

```
master
slave1
slave2
```

### 拷贝Hadoop配置

因为我 Hadoop 使用了 ZooKeeper 高可用模式， HBase 在没有 Hadoop 配置的情况下会找不到 HDFS 的地址。所以需要将配置拷贝到它的目录。

```shell
cp $HADOOP_HOME/etc/hadoop/core-site.xml $HBASE_HOME/conf/
cp $HADOOP_HOME/etc/hadoop/hdfs-site.xml $HBASE_HOME/conf/
```

### 集群同步配置

将配置文件同步到集群中的其他机器中去

```shell
scp -r /usr/local/src/hbase-1.2.0 slave1:/usr/local/src/
scp -r /usr/local/src/hbase-1.2.0 slave2:/usr/local/src/
```

参考 [HBase配置-系统环境变量](https://blackyau.cc/17.html#%E7%B3%BB%E7%BB%9F%E7%8E%AF%E5%A2%83%E5%8F%98%E9%87%8F) 在其他机器中也设置好系统环境变量

### 运行分布式模式

> 在运行之前要先启动 Hadoop ，启动 Hadoop 的命令根据你自己的环境而定

```shell
start-hbase.sh
```

使用 `jps` 查看正在运行的进程是否存在 `HMaster` 和 `HRegionServer`

```shell
[root@master ~]# jps
11538 HMaster
10692 DataNode
10884 NodeManager
10517 JournalNode
11221 DFSZKFailoverController
12008 Jps
11657 HRegionServer
10795 ResourceManager
10446 QuorumPeerMain
10574 NameNode
```

WEB端

http://master:16010

![WEB端截图](https://st.blackyau.net/blog/17/1.png)

## HBase Shell

启动 HBase Shell

```shell
hbase shell
```

如果出现以下报错

```shell
SLF4J: Class path contains multiple SLF4J bindings.
SLF4J: Found binding in [jar:file:/usr/local/src/hbase-1.2.0/lib/slf4j-log4j12-1.7.5.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: Found binding in [jar:file:/usr/local/hadoop-2.6.0/share/hadoop/common/lib/slf4j-log4j12-1.7.5.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
SLF4J: Actual binding is of type [org.slf4j.impl.Log4jLoggerFactory]
```

是因为 `HBase` 自带的 `Jar` 包和 `Hadoop` 的包有冲突，删除冲突包即可

接下来开始创建 数据库/表 插入数据。注意，创建表的时候如果不指定数据库，表就会被放进 `default` 表中

表结构如下:

<escape>
<table>
  <tr>
    <th rowspan="2">Row Key</th>
    <th colspan="2">inside</th>
    <th colspan="2">outside</th>
  </tr>
  <tr>
    <td>name</td>
    <td>age</td>
    <td>slang</td>
    <td>zh</td>
  </tr>
  <tr>
    <td>1</td>
    <td>tom</td>
    <td>3</td>
    <td>cat</td>
    <td>mao</td>
  </tr>
  <tr>
    <td>2</td>
    <td>jerry</td>
    <td>2</td>
    <td>rat</td>
    <td>laoshu</td>
  </tr>
</table>
</escape>

```sql
create_namespace 'test' --创建数据库
create 'test:emp', 'inside', 'outside' --在test库中创建emp表
list_namespace_tables 'test' --查看test中的表
```

```sql
TABLE
emp
1 row(s) in 0.0130 seconds
```

查看表结构

```sql
desc 'test:emp'
```

```sql
Table test:emp is ENABLED
test:emp
COLUMN FAMILIES DESCRIPTION
{NAME => 'inside', BLOOMFILTER => 'ROW', VERSIONS => '1', IN_MEMORY => 'false', KEEP_DELETED_CELLS => 'FALSE', DATA_BLOCK_ENCODING => 'NONE', TTL => 'FOREVER', COMPRESSION => 'NONE', MIN_VERSIONS => '0', BLOCKCACHE => 'true', BLOCKSIZE => '65536', REPLICATION_SCOPE => '0'}
{NAME => 'outside', BLOOMFILTER => 'ROW', VERSIONS => '1', IN_MEMORY => 'false', KEEP_DELETED_CELLS => 'FALSE', DATA_BLOCK_ENCODING => 'NONE', TTL => 'FOREVER', COMPRESSION => 'NONE', MIN_VERSIONS => '0', BLOCKCACHE => 'true', BLOCKSIZE => '65536', REPLICATION_SCOPE => '0'}
2 row(s) in 0.0410 seconds
```

插入数据

```sql
put 'test:emp', '1', 'inside:name', 'tom'
put 'test:emp', '1', 'inside:age', '3'
put 'test:emp', '1', 'outside:slang', 'cat'
put 'test:emp', '1', 'outside:zh', 'mao'
put 'test:emp', '2', 'inside:name', 'jerry'
put 'test:emp', '2', 'inside:age', '2'
put 'test:emp', '2', 'outside:slang', 'rat'
put 'test:emp', '2', 'outside:zh', 'laoshu'
```

扫描表

```sql
scan 'test:emp'
```

```sql
ROW    COLUMN+CELL

 1     column=inside:age, timestamp=1559458876365, value=3
 1     column=inside:name, timestamp=1559458822328, value=tom
 1     column=outside:slang, timestamp=1559459016767, value=cat
 1     column=outside:zh, timestamp=1559459100544, value=mao
 2     column=inside:age, timestamp=1559459253516, value=2
 2     column=inside:name, timestamp=1559459242949, value=jerry
 2     column=outside:slang, timestamp=1559459336624, value=rat
 2     column=outside:zh, timestamp=1559459347676, value=laoshu
2 row(s) in 0.0190 seconds
```

删除表数据之前要先禁用

```sql
disable 'test:emp' --禁用表
drop 'test:emp' --删除表(先别删,后面导出了再删)
```

退出 `HBase Shell`

```sql
quit
```

## 导入导出数据

使用 `HBase` 自带的类 导出 二进制格式文件

> 如果不加 `file://` 就会导出到 `HDFS` 上面去，如果带了数据库名一定要加 `''` 不然导出的数据是空白。

```shell
hbase org.apache.hadoop.hbase.mapreduce.Export `test:emp` file:///root/emp_out
ls /root/emp_out/
```

```shell
part-m-00000  _SUCCESS
```

> 之前导出文件后一直找不到,结果是因为我启用的 `Hadoop HA` 把这个任务分配给了别的机器，所以不在 `master` 上面。上 http://master:8088 看看也能知道是谁在运行。

![资源管理器 截图](https://st.blackyau.net/blog/17/2.png)

删除表，为导入数据做准备

```shell
hbase shell
```

```sql
disable 'test:emp' --禁用表
drop 'test:emp' --删除表
create 'test:emp', 'inside', 'outside' --新建表(没有同名表无法导入数据)
quit
```

导入之前导出的数据

> 注意你的 `MapReduce` 任务会被分配到那台机器上运行，文件要放对位置。

```shell
hbase org.apache.hadoop.hbase.mapreduce.Import `test:emp` file:///root/emp_out
```

## 参考

[Apache HBase ™ Reference Guide Version 1.2.12](https://hbase.apache.org/1.2/book.html)

[CSDN@奔跑的豆子_ - HBase-单机模式安装](https://blog.csdn.net/y472360651/article/details/79017308)

[博客园@huanlegu0426 - hadoop2.5.1+hbase1.1.2安装与配置](https://www.cnblogs.com/huanlegu0426/p/hbase03.html)

[CSDN@jjshouji - hbase 1.2.6 安装](https://blog.csdn.net/jjshouji/article/details/78054556)

[CSDN@疯子. - 大数据系列之数据库Hbase知识整理（三）Hbase的表结构，基本操作，元数据表meta](https://blog.csdn.net/u011444062/article/details/81138861)

[易白教程@张新发 - HBase教程](https://www.yiibai.com/hbase)

[Stack Overflow@Nanda - How to import/export hbase data via hdfs (hadoop commands)](https://stackoverflow.com/questions/25909132)

[简书@冯宇Ops - HBase入坑须知(一)](https://www.jianshu.com/p/a97fbdb2f03f)

[CSDN@芙兰泣露 - hbase导入导出数据](https://blog.csdn.net/u012882134/article/details/52527469)

[CSDN@Data_IT_Farmer - Hbase表两种数据备份方法-导入和导出示例](https://blog.csdn.net/helloxiaozhe/article/details/80325212)

[CSDN@weixin_4065234 - Hbase数据库的常用操作命令](https://blog.csdn.net/weixin_40652340/article/details/78744518)

[CSDN@niugeblog - hbase 命令详解之namespace与table](https://blog.csdn.net/maligebazi/article/details/79952459)

[3NICE - 解决在Markdown中的表格单元格合并的问题](https://3nice.cc/2018/10/01/markdowntable/)