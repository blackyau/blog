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

## 准备工作

这里我选择版本较老的 HIVE 1.1.0 使用，因为它和我[之前安装的 Hadoop ](https://blackyau.cc/11.html)比较配

因为版本太老了在 Mirror 里面都找不到，只有 Archive 里面才有

```shell
curl -O https://archive.apache.org/dist/hive/hive-1.1.0/apache-hive-1.1.0-bin.tar.gz
tar -xf apache-hive-1.1.0-bin.tar.gz
```

## 配置环境变量

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

## 修改配置文件

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

## 启动

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

## 建表

```sql
create database test; -- 创建数据库
show databases; -- 查询已创建的数据库
use test; -- 使用该库
create table test1(hold string, city string, url string)row format delimited fields terminated by '\t';
-- 新建表，并将 \t 作为字段分隔符
desc test1; -- 查看表结构
quit;
```

## 插入数据

这是我使用的测试数据 [small](https://st.blackyau.net/blog/13/small)

```shell
curl -O https://st.blackyau.net/blog/13/small # 将文件下载到本地
hdfs dfs -put ./small / # 将文件传到HDFS
```

接下来的命令在 HIVE 下进行

```sql
use test; -- 使用 test 库
load data inpath "/small" into table test1; -- 将文件加载进 test1 表中
select * from test1;
select city, url from tesst1;
```

输出数据如下即导入成功

```
OK
白银	by.58.com/
庆阳	qingyang.58.com/
嘉峪关	jyg.58.com/
... 手动省略
宜宾	yb.58.com/
自贡	zg.58.com/
乐山	ls.58.com/
	NULL
Time taken: 0.062 seconds, Fetched: 51 row(s)
```

## 插入 Json 数据

```sql
create table test2(
  name string,
  
)
```
## 未完待续

- HIVE 参数设置
- Sqoop 数据推送

## 参考

[Programming Hive](https://book.douban.com/subject/25791255/)

[StackOverFlow@Jonathan L - java.net.URISyntaxException when starting HIVE](https://stackoverflow.com/questions/27099898/)

[StackOverFlow@user1493140 - SLF4J: Class path contains multiple SLF4J bindings](https://stackoverflow.com/questions/14024756/)

[Hadoop: The Definitive Guide@Tom White](https://item.jd.com/12109713.html)