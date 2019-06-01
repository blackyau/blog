---
layout: post
title: HBase 配置
date: 2019-06-01 18:09:22
updated: 2019-06-01 21:30:40
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

> 在 Hadoop 2.6.x 中的 Hadoop 2.6.1 版本下运行 HBase 可能会导致集群故障和数据丢失，请使用 Hadoop 2.6.1+ 版本

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

> 未完待续

## 参考

[Apache HBase ™ Reference Guide Version 1.2.12](https://hbase.apache.org/1.2/book.html)

[CSDN@奔跑的豆子_ - HBase-单机模式安装](https://blog.csdn.net/y472360651/article/details/79017308)