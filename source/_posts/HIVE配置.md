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

## HIVE 参数设置

以下列出的参数大概是有点用的

```xml
  <property>
    <!--- HIVE 大部分操作都会触发一个 MapReduce 修改该参数后它会尝试使用本地模式 以降低资源消耗-->
    <name>hive.exec.mode.local.auto</name>
    <value>true</value>
  </property>

  <property>
    <!--- 这是 HIVE 加载 JAR 的路径 -->
    <name>hive.aux.jars.path</name>
    <value>/root/apache-hive-1.1.0-bin/lib</value>
  </property>

  <property>
    <!--- 如果启用了 LIMIT 优化，这个用来控制 LIMIT 的最小行取样量 -->
    <name>hive.limit.row.max.size</name>
    <value>100000</value>
    <description>When trying a smaller subset of data for simple LIMIT, how much size we need to guarantee each row to have at least.</description>
  </property>
  <property>
    <!--- 如果启用了 LIMIT 优化，这个用来控制 LIMIT 的最大文件数 -->
    <name>hive.limit.optimize.limit.file</name>
    <value>10</value>
    <description>When trying a smaller subset of data for simple LIMIT, maximum number of files we can sample.</description>
  </property>
  <property>
    <!--- 是否启用 LIMIT 优化，这是对元数据进行抽样统计，有可能输入有用的数据永远不会被处理到。毕竟是抽样 -->
    <name>hive.limit.optimize.enable</name>
    <value>true</value>
    <description>Whether to enable to optimization to trying a smaller subset of data for simple LIMIT first.</description>
  </property>

  <property>
    <!--- 并行执行，一个查询可能会有多个阶段而且这些阶段可能并非完全相互依赖，所以阶段越多job可能就更快完成。同时它会增加对集群的利用率 -->
    <name>hive.exec.parallel</name>
    <value>true</value>
    <description>Whether to execute jobs in parallel</description>
  </property>
  <property>
    <!--- 并行执行最大线程数 -->
    <name>hive.exec.parallel.thread.number</name>
    <value>8</value>
    <description>How many jobs at most can be executed in parallel</description>
  </property>

  <property>
    <!--- 严格模式,如果修改为 strict 那么会禁止3种类型的查询:1.不限制分区查询(不允许用户扫描所有分区,where必须有两个或以上) 2.order by必须要LIMIT 3.限制笛卡积尔查询,多表连接查询应该使用join
    和on -->
    <name>hive.mapred.mode</name>
    <value>nonstrict</value>
    <description>
      The mode in which the Hive operations are being performed.
      In strict mode, some risky queries are not allowed to run. They include:
        Cartesian Product.
        No partition being picked up for a query.
        Comparing bigints and strings.
        Comparing bigints and doubles.
        Orderby without limit.
    </description>
  </property>

  <property>
    <!-- 因为Hive使用输入数据量的大小来确定reducer个数，修改这个数量就可以更改使用reducer的个数 -->
    <name>hive.exec.reducers.bytes.per.reducer</name>
    <value>256000000</value>
    <description>size per reducer.The default is 256Mb, i.e if the input size is 1G, it will use 4 reducers.</description>
  </property>
  <property>
    <!-- 设置一个查询最多可消耗的reducers的量-->
    <name>hive.exec.reducers.max</name>
    <value>1009</value>
    <description>
      max number of reducers will be used. If the one specified in the configuration parameter mapred.reduce.tasks is
      negative, Hive will use this one as the max number of reducers when automatically determine number of reducers.
    </description>
  </property>
```

## Scala 数据推送

先下载程序为配置做准备

```shell
curl -O https://mirrors.tuna.tsinghua.edu.cn/apache/sqoop/1.4.7/sqoop-1.4.7.bin__hadoop-2.6.0.tar.gz
tar -xf sqoop-1.4.7.bin__hadoop-2.6.0.tar.gz
```

### 配置环境变量

```shell
vi /etc/profile
```

在文本下方写入以下配置

```shell
export SQOOP_HOME=/root/sqoop-1.4.7.bin__hadoop-2.6.0
export PATH=$PATH:$SQOOP_HOME/bin
```

```shell
sqoop help
```

当终端输出以下信息时说明你的配置成功了

```shell
Warning: /root/sqoop-1.4.7.bin__hadoop-2.6.0/../hbase does not exist! HBase imports will fail.
Please set $HBASE_HOME to the root of your HBase installation.
Warning: /root/sqoop-1.4.7.bin__hadoop-2.6.0/../hcatalog does not exist! HCatalog jobs will fail.
Please set $HCAT_HOME to the root of your HCatalog installation.
Warning: /root/sqoop-1.4.7.bin__hadoop-2.6.0/../accumulo does not exist! Accumulo imports will fail.
Please set $ACCUMULO_HOME to the root of your Accumulo installation.
Warning: /root/sqoop-1.4.7.bin__hadoop-2.6.0/../zookeeper does not exist! Accumulo imports will fail.
Please set $ZOOKEEPER_HOME to the root of your Zookeeper installation.
19/04/20 18:55:32 INFO sqoop.Sqoop: Running Sqoop version: 1.4.7
usage: sqoop COMMAND [ARGS]

Available commands:
  codegen            Generate code to interact with database records
  create-hive-table  Import a table definition into Hive
  eval               Evaluate a SQL statement and display the results
  export             Export an HDFS directory to a database table
  help               List available commands
  import             Import a table from a database to HDFS
  import-all-tables  Import tables from a database to HDFS
  import-mainframe   Import datasets from a mainframe server to HDFS
  job                Work with saved jobs
  list-databases     List available databases on a server
  list-tables        List available tables in a database
  merge              Merge results of incremental imports
  metastore          Run a standalone Sqoop metastore
  version            Display version information

See 'sqoop help COMMAND' for information on a specific command.
```

如果你没有使用 `HBase、HCatalog、Accumulo、Zookeeper` 你可以忽略它的警告，但是如果你和我一样觉得烦。你可以通过注释相关代码以跳过检查。

```shell
## Moved to be a runtime check in sqoop.
#if [ ! -d "${HBASE_HOME}" ]; then
#  echo "Warning: $HBASE_HOME does not exist! HBase imports will fail."
#  echo 'Please set $HBASE_HOME to the root of your HBase installation.'
#fi

## Moved to be a runtime check in sqoop.
#if [ ! -d "${HCAT_HOME}" ]; then
#  echo "Warning: $HCAT_HOME does not exist! HCatalog jobs will fail."
#  echo 'Please set $HCAT_HOME to the root of your HCatalog installation.'
#fi

#if [ ! -d "${ACCUMULO_HOME}" ]; then
#  echo "Warning: $ACCUMULO_HOME does not exist! Accumulo imports will fail."
#  echo 'Please set $ACCUMULO_HOME to the root of your Accumulo installation.'
#fi
#if [ ! -d "${ZOOKEEPER_HOME}" ]; then
#  echo "Warning: $ZOOKEEPER_HOME does not exist! Accumulo imports will fail."
#  echo 'Please set $ZOOKEEPER_HOME to the root of your Zookeeper installation.'
#fi
```

### 安装 mysql

下载安装包

```shell
curl -O https://mirrors.tuna.tsinghua.edu.cn/mysql/yum/mysql57-community-el7/mysql-community-common-5.7.25-1.el7.x86_64.rpm
curl -O https://mirrors.tuna.tsinghua.edu.cn/mysql/yum/mysql57-community-el7/mysql-community-libs-5.7.25-1.el7.x86_64.rpm
curl -O https://mirrors.tuna.tsinghua.edu.cn/mysql/yum/mysql57-community-el7/mysql-community-client-5.7.25-1.el7.x86_64.rpm
curl -O https://mirrors.tuna.tsinghua.edu.cn/mysql/yum/mysql57-community-el7/mysql-community-server-5.7.25-1.el7.x86_64.rpm
```

卸载自带的 MariaDB 或之前安装的 MySQL

```shell

```

安装

```shell
rpm -ivh mysql-community-common-5.7.25-1.el7.x86_64.rpm
rpm -ivh mysql-community-libs-5.7.25-1.el7.x86_64.rpm
rpm -ivh mysql-community-client-5.7.25-1.el7.x86_64.rpm
rpm -ivh mysql-community-server-5.7.25-1.el7.x86_64.rpm
```

配置

```shell
mysqld --initialize --user=mysql # 使用mysql用户运行MySQL
cat /var/log/mysqld.log # 查看一下生成的临时密码
```

```shell
2019-04-21T01:53:51.675008Z 0 [Warning] TIMESTAMP with implicit DEFAULT value is deprecated. Please use --explicit_defaults_for_timestamp server option (see documentation for more details).
2019-04-21T01:53:52.388438Z 0 [Warning] InnoDB: New log files created, LSN=45790
2019-04-21T01:53:52.484682Z 0 [Warning] InnoDB: Creating foreign key constraint system tables.
2019-04-21T01:53:52.549955Z 0 [Warning] No existing UUID has been found, so we assume that this is the first time that this server has been started. Generating a new UUID: 5075e10a-63d8-11e9-9cd1-000c2937e0f0.
2019-04-21T01:53:52.552680Z 0 [Warning] Gtid table is not ready to be used. Table 'mysql.gtid_executed' cannot be opened.
2019-04-21T01:53:52.583134Z 1 [Note] A temporary password is generated for root@localhost: zeAxOg16AZ!7
```

如上: `zeAxOg16AZ!7` 就是密码

接下来启动 MySQL

```shell
systemctl start mysqld # 启动
systemctl status mysqld # 查看运行状态,看到active (running)就行了
```

```sql
mysql -u root -p # 以root身份登录MySQL,密码就是上面的密码
SET PASSWORD = PASSWORD('your_new_password'); # 修改一下密码
select version();
```

输出以下信息说明配置成功

```sql
+-----------+
| version() |
+-----------+
| 5.7.25    |
+-----------+
1 row in set (0.00 sec)
```

### 导入数据

> 未完待续

## 参考

[Programming Hive](https://book.douban.com/subject/25791255/)

[StackOverFlow@Jonathan L - java.net.URISyntaxException when starting HIVE](https://stackoverflow.com/questions/27099898/)

[StackOverFlow@user1493140 - SLF4J: Class path contains multiple SLF4J bindings](https://stackoverflow.com/questions/14024756/)

[Hadoop: The Definitive Guide@Tom White](https://item.jd.com/12109713.html)

[博客园@junneyang - 三句话告诉你 mapreduce 中MAP进程的数量怎么控制？](https://www.cnblogs.com/junneyang/p/5850440.html)

[博客园@xiaodangshan - centos下RPM安装mysql5.7.13](https://www.cnblogs.com/xiaodangshan/p/7230111.html)