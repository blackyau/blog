---
layout: post
title: Kafka 配置
date: 2019-06-04 18:00:00
updated: 2019-06-04 19:51:20
categories: 大数据
tags: 
    - CentOS
    - Linux
    - Kafka
    - ZooKeeper
urlname: 18
comment: true
---

Kafka 和我之前接触的 {% post_link Flume配置 'Flume' %} 非常相识,不过我关心的是它的搭建方式。

<!-- more -->

## 环境介绍

软件版本如下：

| Program | Version | URL |
| --- | --- | --- |
| System | CentOS-7-x86_64-Minimal-1810 | [TUNA Mirrors](https://mirrors.tuna.tsinghua.edu.cn/centos-vault/centos/7.9.2009/isos/x86_64/) |
| JAVA | jdk-8u211-linux-x64.tar.gz | [Oracle](https://www.oracle.com/cn/java/technologies/javase/javase8u211-later-archive-downloads.html) |
| ZooKeeper | zookeeper-3.4.5.tar.gz | [Apache Archive](http://archive.apache.org/dist/zookeeper/zookeeper-3.4.5/) |
| Kafka | kafka_2.11-1.0.0.tgz | [Apache Archive](http://archive.apache.org/dist/kafka/1.0.0/) |

## 目标

- 正确启动 Kafka
- 完成 生产者(producer) 配置
- 完成 消费者(consumer) 配置

## 基础环境配置

参考 {% post_link Hadoop_HA_搭建 'Hadoop HA 搭建' %} 目前已完成 ZooKeeper 环境搭建

| HostName | broker id | Config Name | IP |
| --- | --- | --- | --- |
| master | 1 | server-1.properties | 192.168.66.128 |
| slave1 | 2 | server-2.properties | 192.168.66.129 |
| slave2 | 3 | server-3.properties | 192.168.66.130 |

### zoo.cfg

zoo.cfg 配置如下

```conf
tickTime=2000
initLimit=10
syncLimit=5

dataDir=/usr/local/src/zookeeper-3.4.5/data
dataLogDir=/usr/local/src/zookeeper-3.4.5/logs

clientPort=2181
server.1=master:2888:3888
server.2=slave1:2888:3888
server.3=slave2:2888:3888
```

### 启动 ZooKeeper

在每台主机上都要执行该命令

```shell
zkServer.sh start
```

执行完毕后查看他们的运行状态

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

当有一台主机处于 `leader` 状态，其他的都处于 `follower` 时即启动成功

## 下载解压

```shell
curl -O http://archive.apache.org/dist/kafka/1.0.0/kafka_2.11-1.0.0.tgz
tar xf kafka_2.11-1.0.0.tgz -C /usr/local/src/
mv /usr/local/src/kafka_2.11-1.0.0 /usr/local/src/kafka
```

## 配置

这里我直接就使用我自己，已经安装好的 `ZooKeeper` 作为服务端不使用 `Kafka` 自带的 `ZooKeeper` ，如果你需要使用自带的可以参考以下文档:

[Kafka Quickstart English](https://kafka.apache.org/quickstart)

[Kafka Quickstart 中文文档](http://kafka.apachecn.org/quickstart.html)

### 编写配置文件

新建 `master` 配置文件

```shell
vi /usr/local/src/kafka/config/server-1.properties
```

写入以下内容

```conf
broker.id=1
listeners=PLAINTEXT://master:9093
log.dir=/root/kafka/logs
zookeeper.connect=master:2181,slave1:2181,slave2:2181
```

新建 `slave1` 配置文件

```shell
vi /usr/local/src/kafka/config/server-2.properties
```

写入以下内容

```conf
broker.id=2
listeners=PLAINTEXT://slave1:9093
log.dir=/root/kafka/logs-2
zookeeper.connect=master:2181,slave1:2181,slave2:2181
```

新建 `slave2` 配置文件

```shell
vi /usr/local/src/kafka/config/server-3.properties
```

写入以下内容

```conf
broker.id=3
listeners=PLAINTEXT://slave2:9093
log.dir=/root/kafka/logs-3
zookeeper.connect=master:2181,slave1:2181,slave2:2181
```

### 同步配置

将程序和配置分发到所有主机上

```shell
scp /usr/local/src/kafka slave1:/usr/local/src/
scp /usr/local/src/kafka slave2:/usr/local/src/
```

### 设置环境变量

```shell
vi ~/.bash_profile
```

在文件底部新增以下内容

```shell
export KAFKA_HOME=/usr/local/src/kafka
PATH=$PATH:$KAFKA_HOME/bin
```

使其生效

```shell
source ~/.bash_profile
```

## 运行

在 `master` 上执行

```shell
kafka-server-start.sh /usr/local/src/kafka/config/server-1.properties
```

在 `slave1` 上执行

```shell
kafka-server-start.sh /usr/local/src/kafka/config/server-2.properties
```

在 `slave2` 上执行

```shell
kafka-server-start.sh /usr/local/src/kafka/config/server-3.properties
```

出现类似以下输出内容说明启动成功了

```shell
[2019-06-04 19:18:04,467] INFO [KafkaServer id=1] started (kafka.server.KafkaServer)
```

## 创建Topic

在一个新的窗口打开 `master` 的 shell 执行

```shell
kafka-topics.sh --create --zookeeper master:2181 --replication-factor 2 --partitions 1 --topic my-replicated-topic
```

--replication-factor 2   #复制两份
--partitions 1 #创建1个分区
--topic #主题为my-replicated-topic

查看所有 topic

```shell
kafka-topics.sh --list --zookeeper master:12181
```

应输出

```shell
my-replicated-topic
```

查看 my-replicated-topic 详细信息

```shell
kafka-topics.sh --describe --zookeeper master:2181 --topic my-replicated-topic
```

应输出

```shell
Topic:my-replicated-topic	PartitionCount:1	ReplicationFactor:2	Configs:
	Topic: my-replicated-topic	Partition: 0	Leader: 2	Replicas: 2,3	Isr: 2,3
```

- “leader”是负责给定分区所有读写操作的节点。每个节点都是随机选择的部分分区的领导者。
- “replicas”是复制分区日志的节点列表，不管这些节点是leader还是仅仅活着。
- “isr”是一组“同步”replicas，是replicas列表的子集，它活着并被指到leader。

> 不知到为啥有两个 Leader ，而且看起来 `master` 好像离线了一样。感觉应该是我设置 `--replication-factor 2` 导致的,也可以能是我刚刚调试的时候没有把这个 `topic` 删干净导致的

## 生产者&消费者

在 `master` 上面执行

```shell
kafka-console-producer.sh --broker-list master:9093 --topic my-replicated-topic
```

输入任意字符

在 `slave2` 上面执行

```shell
kafka-console-consumer.sh --bootstrap-server master:9093 --from-beginning --topic my-replicated-topic
```

看到消息同步出现，即成功。

## 参考

[Apache Kafka](https://kafka.apache.org/)

[Kafka 中文文档](http://kafka.apachecn.org/)

[博客园@Mr.心弦 - Kafka【第一篇】Kafka集群搭建](https://www.cnblogs.com/luotianshuai/p/5206662.html)

[CSDN@运维白菜鹏 - kafka搭建入门（手把手教你搭建）](https://blog.csdn.net/weixin_42207486/article/details/80647802)

[CSDN@360linker - kafka如何彻底删除topic及数据](https://blog.csdn.net/belalds/article/details/80575751)