# ElasticSearch数据的同步方案

数据同步可以解决很多种问题，这次主要是为了解决数据库数据量大，连表查询慢等问题

## 1、方案背景

当订单数据量规模足够大或查询统计足够复杂时，通常会采用MySQL + NoSQL的架构方案，这种方案需要将MySQL中数据同步到其它介质，比如ClickHouse、ES等。订单数据的同步面临着诸多挑战，尤其是异构介质，下面会从”数据同步面临的挑战“和以及”订单数据特点“两个维度来分析

### 1.1考虑因素

性能和稳定性： 订单系统为每个公司的核心业务系统，数据的同步尽量对订单系统无感，换言之，同步双写的方案会影响订单系统的性能和稳定性；
幂等性： 同步RPC和异步消息，都可能产生重复数据，数据落入ES时需要考虑数据去重，保证幂等性；
顺序性： 由于网络的不确定性，有可能数据库中先更新的数据后到达ES，导致最新的数据被旧数据覆盖；
关联性： 通常订单数据会包含很多信息，库表设计时通常会进行垂直拆分，通过订单号进行关联，当进行数据同步时，如何处理这些关联表的数据需要考虑

### **ElasticSearch数据的同步方案**

MySQL数据同步到ES中，大致总结可以分为三种方案：

#### **方案1：监听MySQL的Binlog，分析Binlog将数据同步到ES集群中。**

流程如下：

- 给mysql开启binlog功能
- mysql完成增、删、改操作都会记录在binlog中
- 订单搜索服务基于canal监听binlog变化，实时更新elasticsearch中的内容

#### **方案2：直接通过ES API将数据写入到ES集群中。** 

基本步骤如下：

- 订单搜索服务对外提供接口，用来修改elasticsearch中的数据
- 订单管理服务在完成数据库操作后，直接调用订单搜索服务提供的接口

#### **方案3：异步通知**

流程如下：

- 订单管理服务对mysql数据库数据完成增、删、改后，发送MQ消息
- 订单搜索服务监听MQ，接收到消息后完成elasticsearch数据修改

## 2、由于前期需求描述大概是走方案一

方案1监听binlog，并对binlog进行分析数据同步写入到es，这时又会有多种选择了

- canal
- maxwell
- Databus

这边将对比canal和maxwell的优缺点

### Maxwell

Maxwell是一种开源的MySQL数据库同步工具，它可以将MySQL数据库的binlog转化为JSON格式，并将其发送到消息队列中。Maxwell有以下几个特点：

- -易于使用：Maxwell是非常易于使用和部署的，它只需要简单的配置，就可以轻松实现MySQL数据库的同步。
- -高效：Maxwell将binlog转换为JSON格式，相较于其他同步工具而言，更加高效。

### Canal

Canal是阿里巴巴开发的一款数据库同步工具，它可以实现MySQL数据库的binlog解析和日志抓取。Canal的特点如下：

- -支持数据同步：Canal支持数据实时同步，可以将MySQL的数据实时同步到其他数据库或缓存中。
- -支持数据分发：Canal支持数据分发，可以将MySQL的数据分发到不同的应用中。

### 对比分析

Maxwell和Canal都是很好的MySQL数据库同步工具，具有各自的特点。但是相较于Canal而言，Maxwell更加易于使用和部署，同时支持

更多类型的消息队列和数据输出，因此在一些小型应用中，Maxwell可能更加适合使用。Canal则更加适合于大型的应用场景，可以更好

地支持数据迁移、数据同步和数据分发。

Maxwell 比 Canal 更加轻量级

|对比|Canal|Maxwell|
|:--:|:--:|:--:|
|语言|Java|Java|
|数据格式|格式自由|Json|
|采集数据模式|增量|全量/增量|
|数据落地|定制|支持 kafka 等多种平台|
|HA|支持|支持|

## 3、Maxwell

​        Maxwell 是由美国 Zendesk 开源，用 Java 编写的 MySQL 实时抓取软件。 实时读取 MySQL 二进制日志 Binlog，并生成 JSON 格式的消息，作为生产者发送给 Kafka，Kinesis、 RabbitMQ、Redis、Google Cloud Pub/Sub、文件或其它平台的应用程序。 

官网地址：http://maxwells-daemon.io/

### 3.1、Maxwell 的工作原理

​        Maxwell 的工作原理很简单，就是把自己伪装成 MySQL 的一个 slave，然后以 slave 的身份假装从 MySQL(master)复制数据

### 3.2、MySQL 的 binlog

一些命令提示

```mysql
#查看是否开启binlog
show variables like '%log_bin%';
#查看使用的binlog是哪个文件
show master status;
#查看指定binlog的内容
show binlog events in ‘binlog.000004’;
#查看当前binlog的级别
show variables like 'binlog_format';
#修改全局级别 Row、Statement、Mixed
set globle binlog_format='MIXED'
```

#####  1、binlog 的开启

找到 MySQL 配置文件的位置

 ➢ Linux: /etc/my.cnf 如果/etc 目录下没有，可以通过 locate my.cnf 查找位置 ➢ Windows: \my.ini 

➢ 在 mysql 的配置文件下,修改配置 在[mysqld] 区块，设置/添加 log-bin=mysql-bin 这个表示 binlog 日志的前缀是 mysql-bin，以后生成的日志文件就是 mysql-bin.000001 的文件后面的数字按顺序生成，每次 mysql 重启或者到达单个文件大小的阈值时，新生一个 文件，按顺序编号。

##### 2、binlog 的分类设置

mysql binlog 的格式有三种，分别是 STATEMENT,MIXED,ROW。 在配置文件中可以选择配置 binlog_format= statement|mixed|row

三种格式的区别：

 ◼ statement 语句级，binlog 会记录每次一执行写操作的语句。 相对 row 模式节省空间，但是可能产生不一致性，比如 update test set create_date=now(); 如果用 binlog 日志进行恢复，由于执行时间不同可能产生的数据就不同。 

优点： 节省空间 

缺点： 有可能造成数据不一致。

 ◼ row 行级， binlog 会记录每次操作后每行记录的变化。 

优点：保持数据的绝对一致性。因为不管 sql 是什么，引用了什么函数，他只记录 执行后的效果。

缺点：占用较大空间。 

◼ mixed 混合级别，statement 的升级版，一定程度上解决了 statement 模式因为一些情况 而造成的数据不一致问题。默认还是 statement，在某些情况下，譬如： 当函数中包含 UUID() 时； 包含 AUTO_INCREMENT 字段的表被更新时； 执行 INSERT DELAYED 语句时； 用 UDF 时； 会按照 ROW 的方式进行处理 

优点：节省空间，同时兼顾了一定的一致性。

 缺点：还有些极个别情况依旧会造成不一致，另外 statement 和 mixed 对于需要对 binlog 监控的情况都不方便。

综合上面对比，Maxwell 想做监控分析，选择 row 格式比较合适

###  3.3、Maxwell 使用

####  3.3.1、Maxwell 安装部署

##### 1、安装地址 

（1）Maxwell 官网地址：http://maxwells-daemon.io/ 

（2）文档查看地址：http://maxwells-daemon.io/quickstart/

##### 2、安装部署

![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/02bd2ff3ff23d76f6bf448ea94de0ddc.png)

（1）上传 maxwell-1.29.2.tar.gz 到/opt/software 下

（2）解压 maxwell-1.29.2.tar.gz 的安装包到/opt/module 下

```
tar -zxvf maxwell-1.29.2.tar.gz -C /opt/module/
```

##### 3、 MySQL 环境准备

（1）修改 mysql 的配置文件，开启 MySQL Binlog 设置

```powershell
sudo vim /etc/my.cnf
在[mysqld]模块下添加一下内容
[mysqld]
server_id=1
log-bin=mysql-bin
binlog_format=row
#binlog-do-db=test_maxwell //此处指定需要binlog的库，没有指定就是默认这个mysql下所有库

//并重启 Mysql 服务
sudo systemctl restart mysqld
//登录 mysql 并查看是否修改完成
mysql -uroot -p
show variables like '%binlog%';
```

（2）进入/var/lib/mysql 目录，查看 MySQL 生成的 binlog 文件

注：MySQL 生成的 binlog 文件初始大小一定是 154 字节，然后前缀是 log-bin 参数配 置的，后缀是默认从.000001，然后依次递增。除了 binlog 文件文件以外，MySQL 还会额外 生产一个.index 索引文件用来记录当前使用的 binlog 文件。

##### 4、初始化 Maxwell 元数据库

（1）在 MySQL 中建立一个 maxwell 库用于存储 Maxwell 的元数据

```mysql
mysql> CREATE DATABASE maxwell;
```

（2）设置 mysql 用户密码安全级别

```mysql
mysql> set global validate_password_length=4;
mysql> set global validate_password_policy=0;
```

（3）分配一个账号可以操作该数据库

```mysql
mysql> GRANT ALL ON maxwell.* TO 'maxwell'@'%' IDENTIFIED BY '123456';
```

（4）分配这个账号可以监控其他数据库的权限

```mysql
mysql> GRANT SELECT ,REPLICATION SLAVE , REPLICATION CLIENT ON *.* TO maxwell@'%';
```

（5）刷新 mysql 表权限

```mysql
mysql> flush privileges;
```

##### 5、Maxwell 进程启动

Maxwell 进程启动方式有如下两种：

（1）使用命令行参数启动 Maxwell 进程

```powershell
bin/maxwell --user='maxwell' --password='123456' --host='hadoop102' --producer=stdout
```

--user 连接 mysql 的用户 

--password 连接 mysql 的用户的密码 

--host mysql 安装的主机名 

--producer 生产者模式(stdout：控制台 kafka：kafka 集群)

（2）修改配置文件，定制化启动 Maxwell 进程

##### 6、集成RabbitMQ

（1）Maxwell 配置

首先，在 `config.properties` 文件中设置 Maxwell 的基本配置：

```properties
host=127.0.0.1
port=3306
user=maxwell
password=your_password
producer=rabbitmq
rabbitmq_host=127.0.0.1
rabbitmq_port=5672
rabbitmq_exchange=user_behavior_exchange
```

这里，我们设置 Maxwell 的输出 (`producer`) 为 RabbitMQ，并指定了 RabbitMQ 的主机和交换机

（2）RabbitMQ 设置

创建一个交换机和队列

**启动 Maxwell**

```powershell
maxwell --config config.properties
```

#### 3.3.2、实现案例

（1）运行 maxwell 来监控 mysql 数据更新

```powershell
bin/maxwell --user='maxwell' --password='123456' --host='hadoop102' --producer=stdout
```

（2）向 mysql 的 test_maxwell 库的 test 表插入一条数据，查看 maxwell 的控制台输出

```powershell
mysql> insert into test values(1,'aaa');
//控制台情况
{
 "database": "test_maxwell", --库名
 "table": "test", --表名
 "type": "insert", --数据更新类型
 "ts": 1637244821, --操作时间
 "xid": 8714, --操作 id
 "commit": true, --提交成功
 "data": { --数据
 "id": 1,
 "name": "aaa"
 }
}
```

（3）向 mysql 的 test_maxwell 库的 test 表同时插入 3 条数据，控制台出现了 3 条 json 日志，说明 maxwell 是以数据行为单位进行日志的采集的。

```powershell
mysql> INSERT INTO test VALUES(2,'bbb'),(3,'ccc'),(4,'ddd');
{
    "database":"test_maxwell",
    "table":"test",
    "type":"insert",
    "ts":1637245127,
    "xid":9129,
    "xoffset":0,
    "data":{
            "id":2,
            "name":"bbb"
            }
}
{"database":"test_maxwell","table":"test","type":"insert","ts"
:1637245127,"xid":9129,"xoffset":1,"data":{"id":3,"name":"ccc"
}}
{"database":"test_maxwell","table":"test","type":"insert","ts"
:1637245127,"xid":9129,"commit":true,"data":{"id":4,"name":"dd
d"}}

```

（4）修改 test_maxwell 库的 test 表的一条数据，查看 maxwell 的控制台输出

```powershell
mysql> update test set name='abc' where id =1;
{
 "database": "test_maxwell",
 "table": "test",
 "type": "update",
 "ts": 1637245338,
 "xid": 9418,
 "commit": true,
 "data": { --修改后的数据
 "id": 1,
 "name": "abc"
 },
 "old": { --修改前的数据
 "name": "aaa"
 }
}
```

（5）删除 test_maxwell 库的 test 表的一条数据，查看 maxwell 的控制台输出

```powershell
mysql> DELETE FROM test WHERE id =1;
{
 "database": "test_maxwell",
 "table": "test",
 "type": "delete",
 "ts": 1637245630,
 "xid": 9816,
 "commit": true,
 "data": {
 "id": 1,
 "name": "abc"
 }
}

```

#####  1、监控 Mysql 指定表数据输出控制台

（1）运行 maxwell 来监控 mysql 指定表数据更新

```powershell
bin/maxwell --
user='maxwell' --password='123456' --host='hadoop102' --filter 'exclude: *.*, include:test_maxwell.test' --producer=stdout
```

（2）向 test_maxwell.test 表插入一条数据，查看 maxwell 的监控

```mysql
mysql> insert into test_maxwell.test values(7,'ggg');
{
    "database":"test_maxwell",
    "table":"test",
    "type":"insert",
    "ts":1637247760,
    "xid":11818,
    "commit":true,
    "data":{
            "id":7,
            "name":"ggg"
            }
}
```

（3）向 test_maxwell.test2 表插入一条数据，查看 maxwell 的监控

```mysql
mysql> insert into test1 values(1,'nihao');
本次没有收到任何信息
说明 include 参数生效，只能监控指定的 mysql 表的信息
```

注：还可以设置 include:test_maxwell.*，通过此种方式来监控 mysql 某个库的所有 表，也就是说过滤整个库

##### 2、监控 Mysql 指定表全量数据输出控制台，数据初始化

（1）修改 Maxwell 的元数据，触发数据初始化机制，在 mysql 的 maxwell 库中 bootstrap 表中插入一条数据，写明需要全量数据的库名和表名

```mysql
mysql> insert into maxwell.bootstrap(database_name,table_name) values('test_maxwell','test2');

```

启动 maxwell 进程，此时初始化程序会直接打印 test2 表的所有数据

（3） 当数据全部初始化完成以后，Maxwell 的元数据会变化 is_complete 字段从 0 变为 1 start_at 字段从 null 变为具体时间(数据同步开始时间) complete_at 字段从 null 变为具体时间(数据同步结束时间)

## 4、Canal

原理与maxwell一致，Mysql的开启也不赘述了，可以返回3.1-3.2查看

### 4.1、安装地址

官网下载页面进行下载：https://github.com/alibaba/canal/releases

![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/42bf72f459df60394bf670e0eaa98fe0.png)

### 4.2、安装部署

输入命令，记录文件名和位置

```
show master status;
```

解压canal.deployer-1.1.4.tar.gz，我们可以看到里面有四个文件夹

```
bin  canal.deployer-1.1.5.tar.gz  conf  lib  logs  plugin
```

主要看看conf目录下配置文件：

```
canal.properties   example    logback.xml    metrics  spring
```

接着打开配置文件conf/example/instance.properties，配置信息如下：

```properties
## mysql serverId , v1.0.26+ will autoGen
## v1.0.26版本后会自动生成slaveId，所以可以不用配置
# canal.instance.mysql.slaveId=0
 
# 数据库地址
canal.instance.master.address=127.0.0.1:3306
# binlog日志名称
canal.instance.master.journal.name=mysql-bin.000001
# mysql主库链接时起始的binlog偏移量
canal.instance.master.position=154
# mysql主库链接时起始的binlog的时间戳
canal.instance.master.timestamp=
canal.instance.master.gtid=
 
# username/password
# 在MySQL服务器授权的账号密码
canal.instance.dbUsername=canal
canal.instance.dbPassword=Canal@123456
# 字符集
canal.instance.connectionCharset = UTF-8
# enable druid Decrypt database password
canal.instance.enableDruid=false
 
# table regex .*\\..*表示监听所有表 也可以写具体的表名，用，隔开
canal.instance.filter.regex=.*\\..*
# mysql 数据解析表的黑名单，多个表用，隔开
canal.instance.filter.black.regex=
```

`canal.properties`是启动canal server的配置文件

```properties
canal.destinations = example
# conf root dir
canal.conf.dir = ../conf
# auto scan instance dir add/remove and start/stop instance
canal.auto.scan = true
canal.auto.scan.interval = 5
# set this value to 'true' means that when binlog pos not found, skip to latest.
# WARN: pls keep 'false' in production env, or if you know what you want.
canal.auto.reset.latest.pos.mode = false

canal.instance.tsdb.spring.xml = classpath:spring/tsdb/h2-tsdb.xml
#canal.instance.tsdb.spring.xml = classpath:spring/tsdb/mysql-tsdb.xml

canal.instance.global.mode = spring
canal.instance.global.lazy = false
canal.instance.global.manager.address = ${canal.admin.manager}
#canal.instance.global.spring.xml = classpath:spring/memory-instance.xml
canal.instance.global.spring.xml = classpath:spring/file-instance.xml
#canal.instance.global.spring.xml = classpath:spring/default-instance.xml

#canal.destinations = example就是指定instance实例的查找位置，如果我们一个canal server需要监听多个instance(平时各个业务线的数据库都是独立的如商品product，仓库warehouse)，一个instance监听一个数据库，这是最常见的需求了，这时候我就需要配置多个instance，可以直接把example文件夹拷贝两份，分别用数据库命名新文件夹这样方便我们快速了解该文件夹对应的instance是哪个业务线的。然后就是调整canal.properties
#canal.destinations = product,warehouse
#而每个instance的文件下的instance.properties，适配监听的数据库配置信息 就如上方所示
```

最后在安装包目录下执行以下命令就可以启动了

```powershell
sh bin/startup.sh
```

### 4.3、 Web界面（canal-admin）

但是这种方式每新增一个`instance`，都需要修改配置文件并重启，这样会导致数据同步中断不太友好，而且也没有`canal server`服务的状态监控，着实觉得这框架不够完善。

阿里巴巴也考虑到了这些问题，所以提供了`canal-admin`，`canal-admin`设计上是为`canal`提供整体配置管理、节点运维等面向运维的功能，提供相对友好的`WebUI`操作界面，方便更多用户快速和安全的操作。注意：`canal-admin`有以下限制要求

```
MySQL，用于存储配置和节点等相关数据
canal版本，要求>=1.1.4 (需要依赖canal-server提供面向admin的动态运维管理接口)
```

在官网下载`canal-admin`的安装包解压如下：

```
bin  canal.admin-1.1.5.tar.gz  conf  lib  logs
```

直接来看conf下的文件：

```
application.yml  canal_manager.sql  canal-template.properties  instance-template.properties  logback.xml  public
```

这里看到的就是一个`spring boot`框架开发的web项目啦，`anal_manager.sql`就是`canal-admin`服务所依赖的数据库初始化脚本，我们得去`MySQL`执行，然后修改配置文件`application.yml`

```properties
server:
  port: 8089
spring:
  jackson:
    date-format: yyyy-MM-dd HH:mm:ss
    time-zone: GMT+8
    
spring.datasource:
  address: 10.10.0.10:3306
  database: canal_manager
  username: root
  password: root
  driver-class-name: com.mysql.jdbc.Driver
  url: jdbc:mysql://${spring.datasource.address}/${spring.datasource.database}?useUnicode=true&characterEncoding=UTF-8&useSSL=false
  hikari:
    maximum-pool-size: 30
    minimum-idle: 1
    
canal:
  adminUser: admin
  adminPasswd: admin
```

这里就配置一下前面执行SQL脚本数据库的连接信息即可，当然如果端口`8089`被占用了就改成别的，到时候`canal server`配置对应的就行。在`canal-admin`的目录执行下面命令就能启动了

```
sh bin/startup.sh
```

默认登录用户名密码：admin/123456，成功进入之后

我们可以通过界面管理`canal集群、canal server 、server下的instance`。这样无论是我们修改`instance`的配置还是新增一个

`instance`都不需要去服务器操作并重启服务了，是不是很方便，直接通过界面操作修改、重启即可。

当然还是需要像一开始一样在服务器启动`canal server`的，需要把配置`canal.properties`改成如下：

```properties
# register ip
canal.register.ip =

# canal admin config
canal.admin.manager = 10.10.0.10:8089
canal.admin.port = 11110
canal.admin.user = admin
canal.admin.passwd = 4ACFE3202A5FF5CF467898FC58AAB1D615029441
# admin auto register
canal.admin.register.auto = true
canal.admin.register.cluster =
canal.admin.register.name = 
```

### 4.4、实现案例

`canal`官方没有提供与`spring-boot`框架快速整合的starter，根据官网示例直接使用canal client直连canal server操作

引入依赖：

```xml
<dependency>
     <groupId>com.alibaba.otter</groupId>
     <artifactId>canal.client</artifactId>
     <version>1.1.4</version>
</dependency>
```

也可以引入第三方库：

```xml
<dependency>
     <groupId>com.xpand</groupId>
     <artifactId>starter-canal</artifactId>
     <version>0.0.1-SNAPSHOT</version>
</dependency>
```

**示例**：

```java
package com.yecheng.mystudy.controller;

import com.alibaba.otter.canal.client.CanalConnector;
import com.alibaba.otter.canal.client.CanalConnectors;
import com.alibaba.otter.canal.protocol.CanalEntry;
import com.alibaba.otter.canal.protocol.Message;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

import java.net.InetSocketAddress;
import java.util.List;

@Component
@Slf4j
public class CanalClient {
    private final static int BATCH_SIZE = 1000;
    
    public void run() {
        //建立连接
        CanalConnector connector = CanalConnectors.newSingleConnector(new InetSocketAddress("127.0.0.1", 11111), "example", "root", "root");
        try {
            //打开连接(注意pom文件中引入的canal版本一定要和本机启动的版本保持一致，否则可能会出现连接打开被拒绝的情况)
            connector.connect();
            //配置需要监听的数据表（订阅数据库表,全部表）
            connector.subscribe(".*..*");
            //回滚到未ack的地方，下次fetch的时候，可以从最后一个没有ack的地方拿
            connector.rollback();
            //
            while (true) {
                //获取指定数量的数据
                Message message = connector.getWithoutAck(BATCH_SIZE);
                //获取批量ID
                long batchid = message.getId();
                //获取批量的数量
                int size = message.getEntries().size();
                //如果没有数据，线程睡眠一秒
                if (batchid == -1 || size == 0) {
                    try {
                        Thread.sleep(1000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                } else {
                    //如果有数据则处理数据
                    dataHandle(message.getEntries());
                }
                //进行 batch id 的确认。确认之后，小于等于此 batchId 的 Message 都会被确认。
                connector.ack(batchid);
            }

        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            connector.disconnect();
        }
    }
    
    private void dataHandle(List<CanalEntry.Entry> entrys) throws Exception {
        for (CanalEntry.Entry entry : entrys) {
            if (entry.getEntryType() == CanalEntry.EntryType.TRANSACTIONBEGIN || entry.getEntryType() == CanalEntry.EntryType.TRANSACTIONEND) {
                //开启或者关闭事务的实体类型，跳过
                continue;
            }
            //RowChange对象，包含了一行数据变化的所有特征
            CanalEntry.RowChange rowChange = CanalEntry.RowChange.parseFrom(entry.getStoreValue());
            //获取操作类型：insert/update/delete类型
            CanalEntry.EventType eventType = rowChange.getEventType();
            //判断是否为 DDL语句
            if (rowChange.getIsDdl()) {
                log.info("是DDL语句{}", rowChange.getSql());
            }
            // 根据不同的语句类型，处理不同的业务
            if (eventType == CanalEntry.EventType.INSERT) {
                //是新增语句，业务处理。如果新增的时候数据没有发生变化的情况下，是不会被执行
                log.info("新增数据：库名：{},--表名：{}", entry.getHeader().getSchemaName(), entry.getHeader().getTableName());

            } else if (eventType == CanalEntry.EventType.UPDATE) {
                //是修改语句，业务处理。如果修改的时候是没有修改任何数据的情况下，是不会被执行
                log.info("修改数据：库名：{},--表名：{}", entry.getHeader().getSchemaName(), entry.getHeader().getTableName());

            } else if (eventType == CanalEntry.EventType.DELETE) {
                //是删除语句，业务处理。如果删除的时候是没有数据的情况下，是不会被执行
                log.info("删除数据：库名：{},--表名：{}", entry.getHeader().getSchemaName(), entry.getHeader().getTableName());

            }
        }
    }
}

```

且要注意开启监听

```java
package com.yecheng.mystudy;

import com.yecheng.mystudy.controller.CanalClient;
import com.yecheng.mystudy.im.IMService;
import org.mybatis.spring.annotation.MapperScan;
import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

import javax.annotation.Resource;

@SpringBootApplication
// @EnableCanalClient
@MapperScan("com.yecheng.mystudy.mapper")
public class MystudyApplication implements CommandLineRunner {
    @Resource
    private CanalClient canalClient;
    public static void main(String[] args) throws Exception {
        SpringApplication.run(MystudyApplication.class, args);
        IMService.start();

    }

    @Override
    public void run(String... args) throws Exception {
        //项目启动，执行canal客户端监听,原因实现了CommandLineRunner接口，可在项目启动后执行此段代码
        canalClient.run();
    }
}

```

若是使用第三方库，配置：

```properties
//连接canal
canal:
  client:
    instances:
      example:
        host: 192.168.160.136  //canal所在的ip，我这里是虚拟机ip
        port: 11111 //这是默认端口，可以在配置文件该
        batchSize: 1000
```

```java
/**
 * 事件监听器
 */
@CanalEventListener
public class CanalListener {
 
    @Autowired
    private RabbitTemplate rabbitTemplate;
 
    @Autowired
    private CourseService courseService;
 
    /**
     * 监听 edu_course数据库的course表
     */
    @ListenPoint(schema = "edu_course",table = "course")
    public void handleCourseUpdate(CanalEntry.EventType eventType, CanalEntry.RowData rowData){
        //判断操作的类型
        if("DELETE".equals(eventType.name())){
            //遍历删除前数据行的每一列
            rowData.getBeforeColumnsList().forEach(column -> {
                //获得删除前的ID
                if("id".equals(column.getName())){
                    //发删除消息给处理删除的队列
                    rabbitTemplate.convertAndSend(RabbitMQConfig.COURSE_EXCHANGE,RabbitMQConfig.KEY_COURSE_REMOVE,Long.valueOf(column.getValue()));
                    return;
                }
            });
        }else if("INSERT".equals(eventType.name()) || "UPDATE".equals(eventType.name())){
            //获得插入或更新后的数据
            rowData.getAfterColumnsList().forEach(column -> {
                if("id".equals(column.getName())){
                    //通过id查询课程的完整信息
                    Course course = courseService.getCourseById(Long.valueOf(column.getValue()));
                    //包装到Course对象中，转换为JSON
                    String json = JSON.toJSONString(course);
                    //发送给添加或更新队列
                    rabbitTemplate.convertAndSend(RabbitMQConfig.COURSE_EXCHANGE,RabbitMQConfig.KEY_COURSE_SAVE,json);
                }
            });
        }
    }
}
```

```
@EnableCanalClient 加在启动类上
```

以上就完成了Java客户端的代码。这里不做具体的处理，仅仅是打印，先有个直观的感受。

最后我们开始测试，首先启动MySQL、Canal Server，还有刚刚写的Spring Boot项目。然后创建表：

```mysql
CREATE TABLE `tb_commodity_info` (
  `id` varchar(32) NOT NULL,
  `commodity_name` varchar(512) DEFAULT NULL COMMENT '商品名称',
  `commodity_price` varchar(36) DEFAULT '0' COMMENT '商品价格',
  `number` int(10) DEFAULT '0' COMMENT '商品数量',
  `description` varchar(2048) DEFAULT '' COMMENT '商品描述',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='商品信息表';
```

如果我对表进行增删改，canal随时得到监听
使用第三方依赖，比较简单，就直接发结果了
![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/5bee70cd31a4948c5c32f18bbbe0eaae.png)
使用官方依赖：
![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/fe6b43a69bcc326cf07e67d947c9c8d5.png)
![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/351ef09c38b2fc734e8b0b3821dc47ec.png)
我们可以看到更新前的所有数据，以及更新后的所有数据，通过循环遍历即可取到。最后的结果也是数据库产生变化，就会监听到
![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/a2c3113e24846f739e699ad33d4d55a1.png)

## 参考文献

[RabbitMq和Canal的使用](https://blog.csdn.net/m0_57816620/article/details/130099249)

[超详细的canal使用教程及如何通过Spring Boot整合canal优雅实现缓存一致性解决方案](https://zhuanlan.zhihu.com/p/655011270)

[超详细的Canal入门，看这篇就够了](https://zhuanlan.zhihu.com/p/611755041)

尚硅谷大数据课程-Maxwell

[SpringBoot整合Canal实现数据同步到ElasticSearch](https://blog.csdn.net/weixin_50094173/article/details/131932123)
