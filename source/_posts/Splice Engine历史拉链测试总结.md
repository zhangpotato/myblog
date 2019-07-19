---
title: Splice Engine历史拉链测试总结
date: 2019-03-19 14:25:00
author: Potato
top: true # 如果top值为true，则会是首页推荐文章
# 本文章是否开启mathjax，且需要在主题的_config.yml文件中也需要开启才行
mathjax: false
categories: Splice Engine
tags:
  - Splice Engine优化
---

## 基本环境参数

Splice.olap.shuffle.partitions=1200

Dsplice.spark.yarn.executor.memoryOverhead=6144

Dsplice.spark.executor.memory=10g

splice.olap_server.external=true

splice.olap_server.memory=20480

HBASE_HEAPSIZE=20480

splice.splitBlockSize=32108864

## 1、设置splice.olap.shuffle.partitions

在 hbase的 custom hbase-site中添加splice.olap.shuffle.partitions=400，400数值可进行修改，400是一个相对稳妥的数值，该参数影响在**shuffle**过程中分区的个数，从而影响**task**的数量，如下是在测试10G数据时设置400和1200的参数运行效果：

​										**10G结果对比**

|     比较内容     |    400     |   1200   |
| :--------------: | :--------: | :------: |
|      Split       |  **381s**  | **343s** |
| Minor compaction |     0s     |    0s    |
| Major compaction |     0s     |    0s    |
|       闭链       |  **365s**  | **325s** |
| Minor compaction | 152s(24个) |    0s    |
| Major compaction |     0s     |    0s    |
|       开链       |  **105s**  | **131s** |
| Minor compaction |     0s     |    0s    |
| Major compaction |     0s     |    0s    |
|       总计       |  **851s**  | **793s** |

​										**10G测试结果**

| 比较内容 | SM2.7.0.1907 partitions=400 | SM2.7.0.1907 partitions=600 | SM2.7.0.1907 partitions=1200 |
| :------: | :-------------------------: | :-------------------------: | :--------------------------: |
|  SPLIT   |          123.986s           |          109.756s           |           109.925s           |
|   闭链   |           64.074s           |           54.260s           |           53.399s            |
|   开链   |           57.084s           |           55.067s           |           55.086s            |
|   总计   |          245.126s           |          219.083s           |           218.41s            |

适量的设置该参数可以提升运行速度，但是过度设置splice.olap.shuffle.partitions参数不会减少运行时间反而会增加运行时间，所以该参数需要多次尝试修改，并且针对不同的数据量设置的大小也会不一样。

## 2、创建表的字段类型选择问题

```sql
CREATE TABLE T_TELLER_ROLES_EXT_0214(
	ID			VARCHAR(20),
	STAFF_ID 	VARCHAR(20),
	ROLE_ID		VARCHAR(20)
);
```

尽量避免使用VARCHAR() 类型，在进行字段的比较时速度较慢。

**目前还使用的VARCHAR()类型，下一步尝试使用数字类型。**

## 3、修改splice.splitBlockSize

该参数影响使用多少个task从磁盘中读取数据，默认是**67108864**，可设置为**32108864**，具体大小还需要针对不同的运行情况，建议不要修改。

## 4、insert 语句的编写

```sql
#第一种
INSERT INTO T_TELLER_ROLES_ADD --splice-properties bulkImportDirectory='/data/poc/HIS', useSpark=true, skipSampling=false
SELECT * FROM T_TELLER_ROLES_EXT_0215 EXCEPT SELECT * FROM T_TELLER_ROLES_EXT_0214;
```

```sql
#第二种
INSERT INTO T_TELLER_ROLES_ADD SELECT * FROM T_TELLER_ROLES_EXT_0215 EXCEPT SELECT * FROM T_TELLER_ROLES_EXT_0214;
```

第一种是使用SM **bulk load**的方式向表中插入数据，参数bulkImportDirectory指的是SM有读写权限的文件夹，用于保存运行日志，其余参数可在官方文档中有详细解释。

## 5、Error resin resin.U

这个问题指的是资源使用已满，需要重启HBase，这个目前算是个**BUG**，在**1905**版本以后修复了。

## 6、查询不到表的BUG

在创建表后，使用 show tables 无法查询到表的信息，但是可以对表进行操作，这个问题是系统缓存的问题，重启shell客户端，重新进行创建表即可。

## 7、官网参数的修改

**官网参数为：**

|                  参数名称                   |                            数值                             |
| :-----------------------------------------: | :---------------------------------------------------------: |
|    hbase.hstore.defaultengine.compactor     |    com.splicemachine.compactions.SpliceDefaultCompactor     |
| hbase.hstore.defaultengine.compactionpolicy | com.splicemachine.compactions.SpliceDefaultCompactionPolicy |

**修改后：**

|                       参数名称                        |                            数值                             |
| :---------------------------------------------------: | :---------------------------------------------------------: |
|    hbase.hstore.defaultengine.compactor**.class**     |    com.splicemachine.compactions.SpliceDefaultCompactor     |
| hbase.hstore.defaultengine.compactionpolicy**.class** | com.splicemachine.compactions.SpliceDefaultCompactionPolicy |

## 8、如下参数谨慎设置

hbase.hstore.compaction.max.size设置为Long.MAX_VALUE，设置完之后SM服务会无法启动，所以该参数**不要修改**变动。

## 9、HBase自身的问题

**错误一：**So there may be a TCP socket connection left open in CLOSE_WAIT state. For more details check https://issues.apache.org/jira/browse/HBASE-9393
后面的URL指向这个问题的解决方式

## 10、Hmaster与regionserver

配置集群时，Hmaster与regionserver服务不要放在同一节点上，splice服务要与regionserver放在同一节点上。

## 11、删除表后，在hbase中不会删除，而是会在原来表的位置打标记。

运行CALL SYSCS_UTIL.VACUUM();删除已经被删除的表

详细介绍：https://doc.splicemachine.com/sqlref_sysprocs_vacuum.html

在删除一张表并且重新创建一张与原来相同表名的表需要在创建表之前执行该命令。

```sql
Drop table A;
CALL SYSCS_UTIL.VACUUM();
Create table A;
```

## 12、增加spark的执行内存

```shell
Job aborted due to stage failure: Task 15 in stage 181.0 failed 4 times, most recent failure: Lost task 15.3 in stage 181.0 (TID 3633, master, executor 361): ExecutorLostFailure (executor 361 exited caused by one of the running tasks) Reason: Container killed by YARN for exceeding memory limits. 14.3 GB of 14 GB physical memory used. Consider boosting spark.yarn.executor.memoryOverhead.
Driver stacktrace:
```

当出现以上问题时，需要修改如下参数：

Dsplice.spark.yarn.executor.memoryOverhead=6144

Dsplice.spark.executor.memory=10g。

## 13、修改HBASE_HEAPSIZE

官网建议将splice.olap_server.memory的大小设置和HMaster heap size的大小相同。

[OLAP参数设置]: https://doc.splicemachine.com/bestpractices_onprem_configperf.html

## 14、其他优化参数

[提升Splice性能]: https://doc.splicemachine.com/bestpractices_onprem_configperf.html

## 15、向Splice导入数据时注意事项

例：**在所有导入数据的方法中**

```sql
call SYSCS_UTIL.BULK_IMPORT_HFILE (
    schemaName,
    tableName,
    insertColumnList | null,
    fileName,
    columnDelimiter | null,
    characterDelimiter | null,
    timestampFormat | null,
    dateFormat | null,
    timeFormat | null,
    maxBadRecords,
    badRecordDirectory | null,
    oneLineRecords | null,
    charset | null,
    bulkImportDirectory,
    skipSampling
);
```

**bulkImportDirectory**一定要将数据放在**除一级目录**之外的次级目录中，如/data/14g.csv或/data/poc/14g.csv等

[倒入数据方法]: https://doc.splicemachine.com/tutorials_ingest_importoverview.html

在使用**BULK_IMPORT_HFILE**方法时需要修改如下参数：

```shell
yarn.nodemanager.pmem-check-enabled=false
yarn.nodemanager.vmem-check-enabled=false
```

[BULK_IMPORT_HFILE方法]: https://doc.splicemachine.com/tutorials_ingest_importbulkhfile.html

## 16、磁盘管理

当磁盘空间到达阀值时，yarn会拒绝启动，默认阀值为90%，建议不要修改。

[磁盘管理]: https://doc.splicemachine.com/bestpractices_onprem_maintenance.html#DiskSpace

## 17、Splice其他参数

[参数配置]: https://doc.splicemachine.com/bestpractices_onprem_configoptions.html

## 18、查询优化

[查询优化]: https://doc.splicemachine.com/developers_tuning_queryoptimization.html

## 19、导入数据报错

1⃣️

一、连接超时

报错日志如下：

```sh
2019-05-15 13:11:35,679 ERROR [shuffle-client-6-3] server.TransportChannelHandler: Connection to host4/192.168.0.64:7447 has been quiet for 120000 ms while there are outstanding requests. Assuming connection is dead; please adjust spark.network.timeout if this is wrong.
```

解决方法：

```sh
#在hbase-env中添加
export HBASE_MASTER_OPTS="${HBASE_MASTER_OPTS}-Dsplice.spark.shuffle.io.connectionTimeout=480s"
#注意双引号
```

二、无法完成批量加载数据

导入数据命令：

```sh
call SYSCS_UTIL.BULK_IMPORT_HFILE('TPCH', 'ORDERS', null,'/TPCH/1/orders', '|', null, null, null, null, 0,'/TPCH/log/', true, null, '/tmp', false);
```

报错日志如下：

```sh
2019-05-16 17:27:50,481 ERROR [RpcServer.FifoWFPBQ.default.handler=198,queue=18,port=16020] access.SecureBulkLoadEndpoint: Failed to complete bulk load
java.lang.IllegalArgumentException: Wrong FS: hdfs://SM/tmp/5152/650bbcaa243d4b47bc2c1a72973b8599/V/5de6a76d4cb74c3ab50d683166435b4e, expected: hdfs://host0:8020

#当前hdfs的core-site中，fs.defaultFS=SM，所以会报错，期望值为 hdfs://host0:8020
```

解决方法：

```sh
#1、将导入数据命令修改为如下
call SYSCS_UTIL.BULK_IMPORT_HFILE('TPCH', 'ORDERS', null,'/TPCH/1/orders', '|', null, null, null, null, 0,'/TPCH/log/', true, null, 'hdfs://host0:8020/tmp', false);

#2、如果导入数据成功则按照一下步骤进行
#修改hdfs的core-site中fs.defaultFS=hdfs://host0:8020，然后重启服务
fs.defaultFS=hdfs://host0:8020
```

