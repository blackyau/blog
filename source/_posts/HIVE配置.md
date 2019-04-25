---
layout: post
title: Hive 配置
date: 2019-04-19 20:03:50
updated: 2019-04-25 16:18:14
categories: 大数据
tags: 
    - CentOS
    - Linux
    - Hadoop
    - HDFS
    - HIVE
    - MySQL
urlname: 13
comment: true
---

如何将一个基于传统关系型数据库和结构化查询语句（SQL）的现有数据转移到 Hadoop ，对于大量的 SQL 用户来说 HIVE 就是解决这个问题的方案。它提供了一个被称为 Hive 查询语言（简称 HiveQL 或 HQL）的 SQL 方言，来查询储存在 Hadoop 集群中的数据。

<!-- more -->

## 目标

- Hive 建表
- Hive 数据加载
- HQL 编写、数据查询统计
- Sqoop 数据推送 MySQL

## 准备工作

这里我选择版本较老的 HIVE 1.1.0 使用，因为它和我 [之前安装的 Hadoop ](https://blackyau.cc/11.html)比较配

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
    <value/>
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

## 使用 HIVE 导出数据

### 默认参数导出

这条命令是在 `shell` 中执行的，使用 HIVE 的 `-e` 参数，执行完该命令后就直接退出 HIVE，用重定向写进文档。导出的字段是用 是制表符 `\t` 分割的

```shell
hive -e 'use data;select * from tenxun' > /home/hadoop/out
```

下面这条是 `HQL` 命令，它将查询的结果写入到指定的 `local directory` 中去，导出格式和上面的格式相同。

```sql
nsert overwrite local directory '/home/hadoop/out2'
select * from tenxun;
```

### 导出 csv 文件

一开始都和上面一样，后面用 `sed` 将输出的 `\t` 全都都替换为 `,` 后面的 `g` 表示替换所有

```shell
hive -e 'use data;select * from tenxun' | sed 's/[\t]/,/g' > /home/hadoop/out5.csv
```

同时将字段之间的分割符改为 `,` 

```sql
insert overwrite local directory '/home/hadoop/out4'
  row format delimited fields terminated by ','
  select * from tenxun;
```

## Sqoop 数据推送

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

`vi /root/sqoop-1.4.7.bin__hadoop-2.6.0/bin/configure-sqoop`

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

卸载自带的 MariaDB 或之前安装的 MySQL ,如果执行下面的命令存在已安装的包。就用 `rpm -e mysql-community-xxxx` 之类的命令卸载就行了

```shell
rpm -qa | grep mysql
rpm -qa | grep mariadb
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

```sql
use mysql;
update user set host = '%' where user = 'root'; -- 修改允许任何IP使用root身份登录
```

#### 修改 MySQL 配置

打开 MySQL 配置文件

```shell
vi /etc/my.cnf
```

在 `[mysqld]` 字段里加入 `character-set-server=utf8` 如下

```conf
[mysqld]
character-set-server=utf8
bind-address=0.0.0.0
```

在配置文件底部增加以下信息

```conf
[mysql]
no-auto-rehash
default-character-set=utf8

[client]
default-character-set=utf8
```

重启 MySQL

```shell
systemctl restart mysqld
```

登录 MySQL 查看是否修改成功

```sql
mysql -u root -p
show variables like 'char%';
```

输出信息如下说明配置成功

```sql
+--------------------------+----------------------------+
| Variable_name            | Value                      |
+--------------------------+----------------------------+
| character_set_client     | utf8                       |
| character_set_connection | utf8                       |
| character_set_database   | utf8                       |
| character_set_filesystem | binary                     |
| character_set_results    | utf8                       |
| character_set_server     | utf8                       |
| character_set_system     | utf8                       |
| character_sets_dir       | /usr/share/mysql/charsets/ |
+--------------------------+----------------------------+
8 rows in set (0.01 sec)
```

#### 创建用于接收数据的数据库

```sql
create database target;
use target;
CREATE TABLE data (
  name VARCHAR(1000), 
  url VARCHAR(1000), 
  tag VARCHAR(1000), 
  location VARCHAR(1000), 
  release_time VARCHAR(1000), 
  quantity VARCHAR(1000), 
  duties VARCHAR(1000), 
  claim VARCHAR(1000));
```

### 导入测试数据

下载测试数据并上传至 HDFS

```shell
curl -O https://st.blackyau.net/blog/13/testdata
hdfs dfs -put ./testdata /testdata
hive # 进入Hive
```

在Hive中创建数据库

```sql
create database testdata;
```

创建表

```sql
use testdata; -- 使用该数据库
CREATE TABLE test1 (
  `name` string, 
  `url` string, 
  `tag` string, 
  `location` string, 
  `release_time` string, 
  `quantity` int, 
  `duties` string, 
  `claim` string)
ROW FORMAT DELIMITED 
  FIELDS TERMINATED BY '\t'; -- 设置分割符为制表符
```

载入数据

```sql
load data inpath '/testdata' into table test1;
```

输出以下信息时说明导入成功

```sql
Loading data to table testdata.test1
Table testdata.test1 stats: [numFiles=1, totalSize=3401593]
OK
Time taken: 0.414 seconds
```

查询一下

```sql
select * from test1;
select * from test1 limit 10;
```

输出以下信息说明正常

```sql
OK
PCG10-浏览器功能前端开发工程师	https://hr.tencent.com/position_detail.php?id=49321&keywords=&tid=0&lid=0	技术类	深圳	2019-04-11	1	负责QQ浏览器移动端/PC端页面和小程序的功能开发和维护;负责浏览器功能的开发和持续优化；负责浏览器功能的web前端页面开发，维护和优化工作，包括前端JS、HTML5以及nodejs等，持续优化前端页面体验和访问速度;	"2年以上前端开发工作经验；  对浏览器兼容性、前端安全防范、响应式布局、网络协议优化有实践经验；具备良好的沟通能力和团队协作精神
21227-创新手游产品—市场和平台渠道推广	
.......
完善现有运营安全流程及规范;4、持续完善安全组件的监控告警、故障排查5、负责安全平台的网站建设工作	1. 3年以上运维开发及运维平台建设经验；2. 有安全相关行业工作经验；3. 具备良好的沟通能力与项目管理能力；4. 熟悉linux下网络编程，熟悉HTML/JS以及HTTP原理，熟悉多种UI框架5. 精通Python、Shell或其他编程语言，有C、Java或PHP开发经验者优先6. 具备良好的合作精神和快速学习能力；7.有互联网安全运维工作经验者优先。
Time taken: 0.073 seconds, Fetched: 10 row(s)
```

使用较复杂的命令（Hive会调用mapreduce）

```sql
select location,count(*) as temp_sum from test1 group by location order by temp_sum desc;
```

输出以下信息说明工作正常

```sql
Automatically selecting local only mode for query
Query ID = root_20190421152828_c7eaeae7-4e76-48fb-9780-135262ef66e0
Total jobs = 2
Launching Job 1 out of 2
Number of reduce tasks not specified. Estimated from input data size: 1
In order to change the average load for a reducer (in bytes):
  set hive.exec.reducers.bytes.per.reducer=<number>
In order to limit the maximum number of reducers:
  set hive.exec.reducers.max=<number>
In order to set a constant number of reducers:
  set mapreduce.job.reduces=<number>
Job running in-process (local Hadoop)
2019-04-21 15:28:21,976 Stage-1 map = 100%,  reduce = 100%
Ended Job = job_local1573803716_0003
Launching Job 2 out of 2
Number of reduce tasks determined at compile time: 1
In order to change the average load for a reducer (in bytes):
  set hive.exec.reducers.bytes.per.reducer=<number>
In order to limit the maximum number of reducers:
  set hive.exec.reducers.max=<number>
In order to set a constant number of reducers:
  set mapreduce.job.reduces=<number>
Job running in-process (local Hadoop)
2019-04-21 15:28:23,199 Stage-2 map = 100%,  reduce = 100%
Ended Job = job_local1403411610_0004
MapReduce Jobs Launched: 
Stage-Stage-1:  HDFS Read: 20435416 HDFS Write: 1483 SUCCESS
Stage-Stage-2:  HDFS Read: 20436422 HDFS Write: 2021 SUCCESS
Total MapReduce CPU Time Spent: 0 msec
OK
深圳	2128
北京	610
上海	209
广州	123
成都	40
武汉	20
杭州	20
马来西亚	10
香港	10
韩国	10
美国	10
日本	10
Time taken: 2.78 seconds, Fetched: 12 row(s)
```

### 使用 Sqoop 导出数据到 MySQL

下载 [mysql-connector-java-6.0.3](https://st.blackyau.net/blog/13/mysql-connector-java-6.0.3.jar) 并放置在 `$SQOOP_HOME/lib/` 目录下，否则会出现以下报错。

```shell
ERROR sqoop.Sqoop: Got exception running Sqoop: java.lang.RuntimeException: Could not load db driver class: com.mysql.jdbc.Driver
java.lang.RuntimeException: Could not load db driver class: com.mysql.jdbc.Driver
```

下载并保存到指定目录

```shell
curl -o $SQOOP_HOME/lib/mysql-connector-java-6.0.3.jar https://st.blackyau.net/blog/13/mysql-connector-java-6.0.3.jar
```

使用 Sqoop 导出数据到 MySQL 

```shell
sqoop export --connect jdbc:mysql://127.0.0.1/target --driver com.mysql.jdbc.Driver --username root --password SKKfgHfz2AgC --table data --export-dir /user/hive/warehouse/testdata.db/test1 --input-fields-terminated-by '\t'
```

参数解释

`export` ：从HDFS目录导出到数据库表

`--connect`：指定 JDBC 连接参数(在这里就是导出目的地 MySQL 的地址)

`--driver`：指定 JDBC 驱动程序类(不指定的话无法导出到 MySQL ,之前下载的 jar 就是为这个用的)

`--username`：验证用户名

`--password`：验证密码

`--table`：要写入的表名

`--export-dir`：要导出 HDFS 数据的所在源路径(可在 Hive 中通过 `show create table test1` 的 `LOCATION` 看到)

`--input-fields-terminated-by`：指定字段分隔符(我这里的源数据分隔符为'\t')

查询是否导出成功

```sql
use target;
select * from data limit 1;
select name,location,tag from data limit 10;
select location,count(*) as sum from data group by location order by sum desc;
```

输出以下内容说明成功

|  name | location | tag |
|  ---- | -------- | ----|
| 26787-策略分析经理（深圳）                                      | 深圳     | 市场类    |
| 25663-泛互联网售中架构师（北/上/深）                            | 上海     | 技术类    |
| TEG06-Mac终端安全运营高级工程师（深圳）                         | 深圳     | 技术类    |
| 15618-游戏客户端开发工程师（上海）                              | 上海     | 技术类    |
| PCG04-腾讯视频移动终端测试开发工程师（深圳）                    | 深圳     | 技术类    |
| 26787-游戏合作商务                                              | 深圳     | 市场类    |
| TEG06-Mac终端安全运营高级工程师（深圳）                         | 深圳     | 技术类    |
| 15618-游戏客户端开发工程师（上海）                              | 上海     | 技术类    |
| PCG04-腾讯视频移动终端测试开发工程师（深圳）                    | 深圳     | 技术类    |
| 26787-游戏合作商务                                              | 深圳     | 市场类    |

10 rows in set (0.00 sec)


| location | sum  |
| -------- | ---- |
| 深圳         | 2128 |
| 北京         |  610 |
| 上海         |  209 |
| 广州         |  123 |
| 成都         |   40 |
| 武汉         |   20 |
| 杭州         |   20 |
| 日本         |   10 |
| 香港         |   10 |
| 美国         |   10 |
| 马来西亚     |   10 |
| 韩国         |   10 |

12 rows in set (0.00 sec)


## S/HQL 常用命令

```sql
DROP DATABASE test CASCADE; -- 删除 test 数据库并删除里面的数据
show create table test1; -- 查看 test1 表的详细信息
```

## [大数据系列文章](https://blackyau.cc/categories/%E5%A4%A7%E6%95%B0%E6%8D%AE/)

- Hadoop 伪分布部署: https://blackyau.cc/11.html

- Flume 配置: https://blackyau.cc/12.html

- Hive 配置: https://blackyau.cc/13.html

- Hadoop 完全分布部署: https://blackyau.cc/14.html

## 参考

[Programming Hive](https://book.douban.com/subject/25791255/)

[StackOverFlow@Jonathan L - java.net.URISyntaxException when starting HIVE](https://stackoverflow.com/questions/27099898/)

[StackOverFlow@user1493140 - SLF4J: Class path contains multiple SLF4J bindings](https://stackoverflow.com/questions/14024756/)

[Hadoop: The Definitive Guide@Tom White](https://item.jd.com/12109713.html)

[博客园@junneyang - 三句话告诉你 mapreduce 中MAP进程的数量怎么控制？](https://www.cnblogs.com/junneyang/p/5850440.html)

[博客园@xiaodangshan - centos下RPM安装mysql5.7.13](https://www.cnblogs.com/xiaodangshan/p/7230111.html)

[Sqoop User Guide - Connecting to a Database Server](http://sqoop.apache.org/docs/1.4.0-incubating/SqoopUserGuide.html#id1763114)

[StackOverFlow@malatesh - Sqoop: Could not load mysql driver exception](https://stackoverflow.com/questions/22741183/)

[CSDN@一介那个书生 - CentOS下修改mysql数据库编码为UTF-8](https://blog.csdn.net/qq_32953079/article/details/54629245)

[CSDN@爱笑的T_T - CentOS(Linux)中解决MySQL中文乱码](https://blog.csdn.net/u014695188/article/details/51087456)

[StackOverFlow@user1922900 - How to export a Hive table into a CSV file?](https://stackoverflow.com/questions/17086642/)