---
layout: post
title: Storm 配置
date: 2019-06-06 20:50:00
updated: 2019-06-06 21:31:10
categories: 大数据
tags: 
    - CentOS
    - Linux
    - Storm
    - ZooKeeper
urlname: 19
comment: true
---

Storm 是一个分布式实时计算系统，这应该是我最近遇过的搭建最简单的服务了

<!-- more -->

## 环境介绍

软件版本如下：

| Program | Version | URL |
| --- | --- | --- |
| System | CentOS-7-x86_64-Minimal-1810 | [TUNA Mirrors](https://mirrors.tuna.tsinghua.edu.cn/centos/7.6.1810/isos/x86_64/) |
| JAVA | jdk-8u211-linux-x64.tar.gz | [Oracle](https://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html) |
| ZooKeeper | zookeeper-3.4.5.tar.gz | [Apache Archive](http://archive.apache.org/dist/zookeeper/zookeeper-3.4.5/) |
| Storm | apache-storm-1.0.4.tar.gz | [Apache Archive](https://archive.apache.org/dist/storm/apache-storm-1.0.4/) |

## 目标

- 正确启动 Storm
- 有冗余

## 基础环境配置

参考 [Hadoop HA 搭建](https://blackyau.cc/16.html) 目前已完成 ZooKeeper 环境搭建

| HostName | IP |
| --- | --- |
| master | 192.168.66.128 |
| slave1 | 192.168.66.129 |
| slave2 | 192.168.66.130 |

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
curl -O https://archive.apache.org/dist/storm/apache-storm-1.0.4/apache-storm-1.0.4.tar.gz
tar xf apache-storm-1.0.4.tar.gz -C /usr/local/src/
mv /usr/local/src/apache-storm-1.0.4 /usr/local/src/kafka
```

## 配置

Storm 的配置还是挺简单的，因为配置文件就只有一个

```shell
vi /usr/local/src/storm/conf/storm.yaml
```

修改配置文件如下

```yaml
storm.zookeeper.servers:
     - "master"
     - "slave1"
     - "slave2"

nimbus.seeds: ["master", "slave1"]

supervisor.slots.ports:
    - 6700
    - 6701
    - 6702
    - 6703
```

需要留意下 `yaml` 配置文件的格式，和我之前经常见的 `json` 和 `xml` 不太一样。不过因为 `hexo` 配置文件也是用的这个格式，所以感觉还行

## 运行

在 `master` 上执行

```shell
cd /usr/local/src/kafka
./storm nimbus &
./storm ui
```

在 `slave1` 上执行

```shell
./storm nimbus &
```

在 `slave2` 上执行

```shell
./storm supervisor &
```

因为这个程序在运行的时候不会在控制台显示 Log 还不会自动到后台去，就后面加个 `&` 让它到后台去运行了，也可以使用 `ctrl z` 吧它放到后台去。使用 `fg` 可以把它拉到前台，当后台有多个任务的时候用 `jobs` 看看有哪些程序，然后用 `fg id` 把它拉起就行了。 

## 参考

[Storm 1.0.6 Documentation](http://storm.apache.org/releases/1.0.6/index.html)

[CSDN@奔跑-起点 - storm1.x支持主节点nimbus高可用 多master集群部署](https://blog.csdn.net/bbaiggey/article/details/77017230)

[CSDN@Lnho - CentOS下Storm 1.0.0集群安装详解](https://blog.csdn.net/lnho2015/article/details/51143726)