---
layout: post
title: Hive 配置
date: 2019-04-19 20:03:50
updated: 2019-04-19 20:52:37
categories: 大数据
tags: 
    - CentOS
    - Linux
    - Hadoop
    - HDFS
    - HIVE
urlname: 13
comment: true
---

如何将一个基于传统关系型数据库和结构化查询语句（SQL）的现有数据转移到 Hadoop ，对于大量的 SQL 用户来说 HIVE 就是解决这个问题的方案。它提供了一个被称为 Hive 查询语言（简称 HiveQL 或 HQL）的 SQL 方言，来查询储存在 Hadoop 集群中的数据。

<!-- more -->

## 0 准备工作

这里我选择版本较老的 HIVE 1.1.0 使用，因为它和我[之前安装的 Hadoop ](https://blackyau.cc/11.html)比较配

因为版本太老了在 Mirror 里面都找不到，只有 Archive 里面才有

```shell
curl -O https://archive.apache.org/dist/hive/hive-1.1.0/apache-hive-1.1.0-bin.tar.gz
tar -xf apache-hive-1.1.0-bin.tar.gz
```

## 1 配置环境变量

```shell
vi /etc/profile
```

在配置下方添加以下内容

```conf
export HIVE_HOME=/root/apache-hive-1.1.0-bin
export PATH=$PATH:$HIVE_HOME/bin
```

让系统更新配置文件

```shell
source /etc/profile
```

## 2 修改配置文件

```shell
cp /root/apache-hive-1.1.0-bin/conf/hive-default.xml.template /root/apache-hive-1.1.0-bin/conf/hive-site.xml # 使用默认配置文件为正式配置文件
vi /root/apache-hive-1.1.0-bin/conf/hive-site.xml
```

在文件头部增加以下信息，如果不加启动会[报错](https://stackoverflow.com/questions/27099898/java-net-urisyntaxexception-when-starting-hive)

```xml
<property>
    <name>system:java.io.tmpdir</name>
    <value>/root/apache-hive-1.1.0-bin/tmp</value>
  </property>
  <property>
    <name>system:user.name</name>
    <value>root</value>
  </property>
  <property>
```

## 3 启动

```shell
hive
```

如果你的的命令行变为了

```shell
hive> 
```

说明 HIVE 已经启动成功了

如果你遇到了这类错误

```shell
SLF4J: Class path contains multiple SLF4J bindings.
```

可以执行以下指令，删除冲突的 jar 包

```shell
rm /root/apache-hive-1.1.0-bin/lib/hive-jdbc-1.1.0-standalone.jar
```

## 未完待续

- HIVE 建表
- HIVE 数据加载
- HIVE 参数设置
- Sqoop 数据推送