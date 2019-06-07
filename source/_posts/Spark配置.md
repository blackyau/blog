---
layout: post
title: Spark 配置
date: 2019-06-07 14:34:34
updated: 2019-06-07 17:21:15
categories: 大数据
tags: 
    - CentOS
    - Linux
    - Hadoop
    - Spark
    - YARN
urlname: 20
comment: true
---

`Apache Spark` 是一个开源集群运算框架，作用类似于 `Hadoop` 的 `MapReduce` 。但是相对于 `MapReduce` 来说它的速度要快得多。
<!-- more -->

## 环境介绍

软件版本如下：

| Program | Version | URL |
| --- | --- | --- |
| System | CentOS-7-x86_64-Minimal-1810 | [TUNA Mirrors](https://mirrors.tuna.tsinghua.edu.cn/centos/7.6.1810/isos/x86_64/) |
| JAVA | jdk-8u211-linux-x64.tar.gz | [Oracle](https://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html) |
| Hadoop | hadoop-2.6.0.tar.gz | [Apache Archive](http://archive.apache.org/dist/hadoop/common/hadoop-2.6.0/) |
| Spark | spark-2.0.0-bin-hadoop2.6.gz | [Apache Archive](http://archive.apache.org/dist/spark/spark-2.0.0/) |
| ZooKeeper | zookeeper-3.4.5.tar.gz | [Apache Archive](http://archive.apache.org/dist/zookeeper/zookeeper-3.4.5/) |

## 目标

- 完成 Standalone 集群搭建
- 在 YARN 上运行 Spark
- 在 Mesos 上运行 Spark

## 基础环境配置

参考 [Hadoop HA 搭建](https://blackyau.cc/16.html) 目前已完成 Hadoop/ZooKeeper 环境搭建

### 下载解压

```shell
curl -O http://archive.apache.org/dist/spark/spark-2.0.0/spark-2.0.0-bin-hadoop2.6.tgz
tar xf spark-2.0.0-bin-hadoop2.6.tgz -C /usr/local/src/
mv /usr/local/src/spark-2.0.0-bin-hadoop2.6 /usr/local/src/spark
```

## Standalone Mode

使用这种模式搭建，不需要借助其他外部工具(高可用性需要 ZooKeeper)

### 手动启动集群

先搭建一个最简单的

| HostName | Mode | IP |
| --- | --- | --- |
| master | Master | 192.168.66.128 |
| slave1 | Worker | 192.168.66.129 |
| slave2 | Worker | 192.168.66.130 |

不需要修改任何配置，直接启动即可。

```shell
# master
cd /usr/local/src/spark/sbin
./start-master.sh
```

用浏览器打开的 `Web UI` 看看 http://master:8080/

其中的 `URL` 是让其他的 `Workers` 连接到 `master` 的重要参数

![1](https://st.blackyau.net/blog/20/1.png)

将程序传到另外两台机子上

```shell
scp -r /usr/local/src/spark slave1:/usr/local/src/
scp -r /usr/local/src/spark slave2:/usr/local/src/
```

接下来在另外两台机子上启动 `workers` 并连接到 `master`

```shell
# 两台机子都要运行
cd /usr/local/src/spark/sbin
./start-slave.sh spark://master:7077
```

如图出现了两个 `Worker` 且 `State` 处于 `ALIVE` 搭建完毕

![2](https://st.blackyau.net/blog/20/2.png)

### 脚本启动集群

首先将之前启动的服务都手动关掉

```shell
# master
./stop-master.sh
```

```shell
# 两个 slave 都需要关闭
./stop-slave.sh
```

将默认的配置文件改名为正式使用

```shell
cd /usr/local/src/spark/conf/
cp spark-env.sh.template spark-env.sh
cp slaves.template slaves
```

修改 `slaves` 文件删掉里面的所有内容写入以下内容

```
slave1
slave2
```

修改 `spark-env.sh` 文件指定 `master`，写入以下内容

```sh
SPARK_MASTER_HOST=master
JAVA_HOME=/usr/local/src/jdk1.8.0_211
```

> 如果不写 `JAVA_HOME` 的话，在启动 `slave` 的时候会报 `JAVA_HOME is not set` 估计是因为我的 `JAVA_HOME` 设置的仅对 `root` 生效的原因吧

将配置文件同步给另外两台机子

```shell
scp -r /usr/local/src/spark/conf/ slave1:/usr/local/src/spark/
scp -r /usr/local/src/spark/conf/ slave2:/usr/local/src/spark/
```

在 `master` 上面启动 `master` 和 `slaves`

```shell
cd /usr/local/src/spark/sbin/
./start-master.sh
./start-slaves.sh
```

启动成功后浏览器打开 http://master:8080/ 看看，应该和上面的图片是一样的

下面的命令可以关闭

```shell
./stop-master.sh
./stop-slaves.sh
```

### 高可用

编辑配置文件

```shell
vi /usr/local/src/spark/conf/spark-env.sh
```

在配置文件中新增以下内容

```shell
SPARK_DAEMON_JAVA_OPTS="-Dspark.deploy.recoveryMode=ZOOKEEPER -Dspark.deploy.zookeeper.url=master:2181,slave1:2181,slave2:2181 -Dspark.deploy.zookeeper.dir=/sparkha"
```

> 注意如果你之前添加了 `SPARK_MASTER_HOST=master` 要删掉，因为 `master` 的任命被 `ZooKeeper` 接管了这个配置没用了

将配置同步到另外两台机器上

```shell
scp -r /usr/local/src/spark/conf/ slave1:/usr/local/src/spark/
scp -r /usr/local/src/spark/conf/ slave2:/usr/local/src/spark/
```

启动 `ZooKeeper`，如果你还没有配置可以参考这篇文章[Hadoop HA 配置 - ZooKeeper 配置](https://blackyau.cc/16.html#ZooKeeper-%E9%85%8D%E7%BD%AE)

```shell
# 每台机子都要启动
zkServer.sh start
```

在 `mastrt` 上启动 `master` 和 `slaves`

```shell
./start-master.sh
./start-slaves.sh
```

在 `slave1` 上启动 `备用 master`

```shell
./start-master.sh
```

![3](https://st.blackyau.net/blog/20/3.png)

## 参考

[Apache Kafka](https://kafka.apache.org/)

[Kafka 中文文档](http://kafka.apachecn.org/)

[博客园@Mr.心弦 - Kafka【第一篇】Kafka集群搭建](https://www.cnblogs.com/luotianshuai/p/5206662.html)

[CSDN@运维白菜鹏 - kafka搭建入门（手把手教你搭建）](https://blog.csdn.net/weixin_42207486/article/details/80647802)

[CSDN@360linker - kafka如何彻底删除topic及数据](https://blog.csdn.net/belalds/article/details/80575751)