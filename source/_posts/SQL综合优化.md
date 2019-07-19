---
title: MySQL综合优化
date: 2019-04-11 16:03:00
author: Potato
top: true # 如果top值为true，则会是首页推荐文章
# 本文章是否开启mathjax，且需要在主题的_config.yml文件中也需要开启才行
mathjax: false
categories: 数据库
tags:
  - MySQL优化
---

## **MySQL如何实现优化？**

### 1、**数据库设计要合理**（3F）

#### 1F **原子约束** 

当关系模式R的所有属性都不能在分解为更基本的数据单位时，称R是满足第一范式的，简记为1NF。简单的来说就是列无法再进行分割。

 ①、每一列属性都是不可再分的属性值，确保每一列的原子性

 ②、两列的属性相近或相似或一样，尽量合并属性一样的列，确保不产生冗余数据。

#### 2F**保证唯一性**

主键 （不能用id作为订单号码，订单号码利用分布式锁提前将订单号生成好，存放在redis中，需要的话直接去redis中获取 ）

每一行的数据只能与其中一列相关，即一行数据只做一件事。只要数据列中出现数据重复，就要把表拆分开来。

#### 3F**不要有冗余数据**

id	name	address	classid	classname

1	a		北京		1		chinese

2	b		北京		2		math

数据不能存在传递关系，即每个属性都跟主键有直接关系而不是间接关系。像：a-->b-->c  属性之间含有这样的关系，是不符合第三范式的。

比如Student表（学号，姓名，年龄，性别，所在院校，院校地址，院校电话）

这样一个表结构，就存在上述关系。 学号--> 所在院校 --> (院校地址，院校电话)

这样的表结构，我们应该拆开来，如下。

（学号，姓名，年龄，性别，所在院校）--（所在院校，院校地址，院校电话）

### 2、**添加索引**

##### **①索引的定义**

　　数据库索引好比是一本书前面的目录，能加快数据库的查询速度。[索引](http://baike.baidu.com/view/262241.htm)是对数据库表中一个或多个列（例如，employee 表的姓氏 (lname) 列）的值进行排序的结构。如果想按特定职员的姓来查找他或她，则与在表中搜索所有的行相比，索引有助于更快地获取信息。

##### **②建立索引的优缺点：**

###### **优点**：

​    1.大大加快数据的检索速度; 
​    2.创建唯一性索引，保证数据库表中每一行数据的唯一性; 
​    3.加速表和表之间的连接; 
​    4.在使用分组和排序子句进行数据检索时，可以显著减少查询中分组和排序的时间。

###### **缺点**：

　　1.索引需要占用数据表以外的物理存储空间

　　2.创建索引和维护索引要花费一定的时间

　　3.当对表进行更新操作时，索引需要被重建，这样降低了数据的维护速度。

##### **③、索引类型**

索引分为聚簇索引和非聚簇索引两种，聚簇索引是按照数据存放的物理位置为顺序的，而非聚簇索引就不一样了；聚簇索引能提高多行检索的速度，而非聚簇索引对于单行的检索很快。

###### **主键索引**：

primary key	数据库表经常有一列或列组合，其值唯一标识表中的每一行，该列称为表的主键。   在数据库关系图中为表定义主键将自动创建主键索引，主键索引是唯一索引的特定类型。该索引要求主键中的每个值都唯一。当在查询中使用主键索引时，它还允许对数据的快速访问。 

###### **唯一索引**：

UNIQUE   表明此索引的每一个索引值只对应唯一的数据记录，对于单列惟一性索引，这保证单列不包含重复的值。

###### **普通索引：**	

这是最基本的索引，它没有任何限制，比如上文中为title字段创建的索引就是一个普通索引，MyIASM中默认的BTREE类型的索引，也是我们大多数情况下用到的索引。

###### **全文索引**

FULLTEXT	索引仅可用于 MyISAM 表；他们可以从CHAR、VARCHAR或TEXT列中作为CREATE TABLE语句的一部分被创建，或是随后使用ALTER TABLE 或CREATE INDEX被添加。对于较大的数据集，将你的资料输入一个没有FULLTEXT索引的表中，然后创建索引，其速度比把资料输入现有FULLTEXT索引的速度更为快。不过切记对于大容量的数据表，生成全文索引是一个非常消耗时间非常消耗硬盘空间的做法。

###### **组合索引**

平时用的SQL查询语句一般都有比较多的限制条件，所以为了进一步榨取MySQL的效率，就要考虑建立组合索引。例如上表中针对title和time建立一个组合索引：ALTER TABLE article ADD INDEX index_titme_time (title(50),time(10))。建立这样的组合索引，其实是相当于分别建立了下面两组组合索引：

#### **MySQL的数据结构**

##### **B树**

B树（B-tree）是一种树状数据结构，它能够存储数据、对其进行排序并允许以O(log n)的时间复杂度运行进行查找、顺序读取、插入和删除的数据结构。B树，概括来说是一个节点可以拥有多于2个子节点的二叉查找树。与自平衡二叉查找树不同，B-树为系统最优化**大块数据的读和写操作**。B-tree算法减少定位记录时所经历的中间过程，从而加快存取速度。普遍运用在**数据库**和**文件系统**。

**B 树**可以看作是对2-3查找树的一种扩展，即他允许每个节点有M-1个子节点。

- 根节点至少有两个子节点
- 每个节点有M-1个key，并且以升序排列
- 位于M-1和M key的子节点的值位于M-1 和M key对应的Value之间
- 其它节点至少有M/2个子节点
- B Tree(B-):2-3树，每个节点上最多保留3个关键码，超过三个则取从小到大第2个位置的关键码，口诀取2留3

下图是一个M=4 阶的B树:

![](/images/MySQL优化/1.png)

可以看到B树是2-3树的一种扩展，他允许一个节点有多于2个的元素。

B树的插入及平衡化操作和2-3树很相似，这里就不介绍了。下面是往B树中依次插入

**6 10 4 14 5 11 15 3 2 12 1 7 8 8 6 3 6 21 5 15 15 6 32 23 45 65 7 8 6 5 4** 	的演示动画：

![](/images/MySQL优化/2.gif)

#####  **B+树**

​    我们经常听到B+树就是这个概念，用这个树的目的和红黑树差不多，也是为了尽量保持树的平衡，当然红黑树是二叉树，但B+树就不是二叉树了，节点下面可以有多个子节点，数据库开发商会设置子节点数的一个最大值，这个值不会太小，所以B+树一般来说比较矮胖，而红黑树就比较瘦高了。
关于B+树的插入，删除，会涉及到一些算法以保持树的平衡，这里就不详述了。ORACLE的默认索引就是这种结构的。
如果经常需要同时对两个字段进行AND查询,那么使用两个单独索引不如建立一个复合索引，因为两个单独索引通常数据库只能使用其中一个，而使用复合索引因为索引本身就对应到两个字段上的，效率会有很大提高。

**B+**树是对B树的一种变形树，它与B树的差异在于：

- 有k个子结点的结点必然有k个关键码；
- 非叶结点仅具有索引作用，跟记录有关的信息均存放在叶结点中。
- 树的所有叶结点构成一个有序链表，可以按照关键码排序的次序遍历全部记录。

如下图，是一个B+树:

![](/images/MySQL优化/3.png)

![](/images/MySQL优化/4.gif)

### 3、**分表分库技术**

取模分表、水平分割、垂直分割

#### **什么是分库分表**？

顾名思义，分库分表就是按照一定的规则，对原有的数据库和表进行拆分，把原本存储于一个库的数据分块存储到多个库上，把原本存储于一个表的数据分块存储到多个表上。

#### **为什么要分库分表**？

随着时间和业务的发展，数据库中的数据量增长是不可控的，库和表中的数据会越来越大，随之带来的是更高的磁盘、IO、系统开销，甚至性能上的瓶颈，而一台服务的资源终究是有限的，因此需要对数据库和表进行拆分，从而更好的提供数据服务。

可以用说用到MySQL的地方,只要数据量一大, 马上就会遇到一个问题,要分库分表. 这里引用一个问题为什么要分库分表呢?MySQL处理不了大的表吗? 其实是可以处理的大表的.我所经历的项目中单表物理上文件大小在80G多,单表记录数在5亿以上,而且这个表 属于一个非常核用的表:朋友关系表. 

但这种方式可以说不是一个最佳方式. 因为面临文件系统如Ext3文件系统对大于大文件处理上也有许多问题. 这个层面可以用xfs文件系统进行替换.但MySQL单表太大后有一个问题是不好解决: 表结构调整相关的操作基 
本不在可能.所以大项在使用中都会面监着分库分表的应用. 

从Innodb本身来讲数据文件的Btree上只有两个锁, 叶子节点锁和子节点锁,可以想而知道,当发生页拆分或是添加新叶时都会造成表里不能写入数据.所以分库分表还就是一个比较好的选择了. 

#### **怎么分库分表合适呢**? 

经测试在单表1000万条记录一下,写入读取性能是比较好的. 这样在留点buffer,那么单表全是数据字型的保持在800万条记录以下, 有字符型的单表保持在500万以下. 

如果按 100库100表来规划,如用户业务: 500万*100*100 = 50000000万 = 5000亿记录. 

心里有一个数了,按业务做规划还是比较容易的.

分布式数据库架构--排序、分页、分组、实现

#### **什么时候分库**？

电商项目将一个项目进行拆分，拆成多个小项目，每个小项目都有自己单独的数据库，互不影响（垂直分割订单数据库）

#### **什么时候分表**？

对于某网站平台的数据库表-公司表，数据量很大，这种能预估出来的大数据量表，我们就事先分出个N个表，这个N是多少，根据实际情况而定。

#### **为什么分表**？

当一张表的数据达到几千万时，你查询一次所花的时间会变多，如果有联合查询的话，我想有可能会死在那儿了。分表的目的就在于此，减小数据库的负担，缩短查询时间。

某网站现在的数据量至多是5000万条，可以设计每张表容纳的数据量是500万条，也就是拆分成10张表，

那么如何判断某张表的数据是否容量已满呢？可以在程序段对于要新增数据的表，在插入前先做统计表记录数量的操作，当<500万条数据，就直接插入，当已经到达阀值，可以在程序段新创建数据库表（或者已经事先创建好），再执行插入操作。

##### **垂直分库**/分表

如果把业务切割得足够独立，那把不同业务的数据放到不同的数据库服务器将是一个不错的方案，而且万一其中一个业务崩溃了也不会影响其他业务的正常进行，并且也起到了负载分流的作用，大大提升了数据库的吞吐能力。经过垂直分区后的数据库架构图如下：

![](/images/MySQL优化/5.jpg)

然而，尽管业务之间已经足够独立了，但是有些业务之间或多或少总会有点联系，如用户，基本上都会和每个业务相关联，况且这种分区方式，也不能解决单张表数据量暴涨的问题，因此可以尝试水平分割

###### **优点：**

1. 拆分后业务清晰，达到专库专用。
2. 可以实现热数据和冷数据的分离，将不经常变化的数据和变动较大的数据分散再不同的库/表中。
3. 便于维护

###### **缺点：**

1. 不解决数据量大带来的性能损耗，读写压力依旧很大
2. 不同的业务无法跨库关联（join），只能通过业务来关联

##### **水平分库/分表**

这是一个非常好的思路，将用户按一定规则（按id哈希）分组，并把该组用户的数据存储到一个数据库分片中，即一个sharding，这样随着用户数量的增加，只要简单地配置一台服务器即可，原理图如下：

![](/images/MySQL优化/6.jpg)

如何来确定某个用户所在的shard呢，可以建一张用户和shard对应的数据表，每次请求先从这张表找用户的shard id，再从对应shard中查询相关数据，如下图所示：

![](/images/MySQL优化/7.jpg)

###### **优点：**

1. 单库（表）的数据量得以减少，提高性能
2. 提高了系统的稳定性和负载能力
3. 切分出的表结构相同，程序改动较少

###### **缺点：**

1. 拆分规则较难抽象
2. 数据分片在扩容时需要迁移
3. 维护量增大
4. 依然存在跨库无法join等问题，同时涉及分布式事务，数据一致性等问题。

#### **分库/分表分类**

##### **单库单表** 

单库单表是最常见的数据库设计，例如，有一张用户(user)表放在数据库db中，所有的用户都可以在db库中的user表中查到。 

#####  **单库多表** 

随着用户数量的增加，user表的数据量会越来越大，当数据量达到一定程度的时候对user表的查询会渐渐的变慢，从而影响整个DB的性能。如果使用mysql, 还有一个更严重的问题是，当需要添加一列的时候，mysql会锁表，期间所有的读写操作只能等待。 

可以通过某种方式将user进行水平的切分，产生两个表结构完全一样的user_0000,user_0001等表，user_0000 + user_0001 + …的数据刚好是一份完整的数据。 

##### **多库多表**

随着数据量增加也许单台DB的存储空间不够，随着查询量的增加单台数据库服务器已经没办法支撑。这个时候可以再对数据库进行水平区分。 

#### **分库分表规则**

 设计表的时候需要确定此表按照什么样的规则进行分库分表。例如，当有新用户时，程序得确定将此用户信息添加到哪个表中；同理，当登录的时候我们得通过用户的账号找到数据库中对应的记录，所有的这些都需要按照某一规则进行。 
路由 

通过分库分表规则查找到对应的表和库的过程。如分库分表的规则是user_id mod 4的方式，当用户新注册了一个账号，账号id的123,我们可以通过id mod 4的方式确定此账号应该保存到User_0003表中。当用户123登录的时候，我们通过123 mod 4后确定记录在User_0003中。 

#### **分库分表产生的问题，及注意事项** 

##### **分库分表维度的问题** 

假如用户购买了商品,需要将交易记录保存取来，如果按照用户的纬度分表，则每个用户的交易记录都保存在同一表中，所以很快很方便的查找到某用户的 购买情况，但是某商品被购买的情况则很有可能分布在多张表中，查找起来比较麻烦。反之，按照商品维度分表，可以很方便的查找到此商品的购买情况，但要查找 到买人的交易记录比较麻烦。 

所以常见的解决方式有： 

- 通过扫表的方式解决，此方法基本不可能，效率太低了。
- 记录两份数据，一份按照用户纬度分表，一份按照商品维度分表。
- 通过搜索引擎解决，但如果实时性要求很高，又得关系到实时搜索。 

##### **联合查询的问题** 

联合查询基本不可能，因为关联的表有可能不在同一数据库中。 

##### **避免跨库事务**

避免在一个事务中修改db0中的表的时候同时修改db1中的表，一个是操作起来更复杂，效率也会有一定影响。 

##### **尽量把同一组数据放到同一**DB服务器上

例如将卖家a的商品和交易信息都放到db0中，当db1挂了的时候，卖家a相关的东西可以正常使用。也就是说避免数据库中的数据依赖另一数据库中的数据。 

##### **一主多备**

在实际的应用中，绝大部分情况都是读远大于写。Mysql提供了读写分离的机制，所有的写操作都必须对应到Master，读操作可以在 Master和Slave机器上进行，Slave与Master的结构完全一样，一个Master可以有多个Slave,甚至Slave下还可以挂 Slave,通过此方式可以有效的提高DB集群的 QPS.                                                       

所有的写操作都是先在Master上操作，然后同步更新到Slave上，所以从Master同步到Slave机器有一定的延迟，当系统很繁忙的时候，延迟问题会更加严重，Slave机器数量的增加也会使这个问题更加严重。 
此外，可以看出Master是集群的瓶颈，当写操作过多，会严重影响到Master的稳定性，如果Master挂掉，整个集群都将不能正常工作。 
所以

- 当读压力很大的时候，可以考虑添加Slave机器的分式解决，但是当Slave机器达到一定的数量就得考虑分库了。
- 当写压力很大的时候，就必须得进行分库操作。 

### **4、读写分离**

MySQL的主从复制解决了数据库的读写分离，并很好的提升了读的性能

![](/images/MySQL优化/8.jpg)

其主从复制的过程如下图所示：

![](/images/MySQL优化/9.jpg)

主从复制也带来其他一系列性能瓶颈问题：

1. 写入无法扩展
2. 写入无法缓存
3. 复制延时
4. 锁表率上升
5. 表变大，缓存率下降

##### **使用Sharding-JDBC进行分库分表**

###### **简介**

Sharding-JDBC是一个开源的分布式数据库中间件，它无需额外部署和依赖，完全兼容JDBC和各种ORM框架。Sharding-JDBC作为面向开发的微服务云原生基础类库，完整的实现了分库分表、读写分离和分布式主键功能，并初步实现了柔性事务。关于sj的详细配置和使用方法请参见[官方文档](https://link.jianshu.com?t=http%3A%2F%2Fshardingsphere.io%2Fdocs_cn%2F00-overview%2F)

###### **配置**

Sharing-JDBC的springboot starter对springboot 2.0还不支持。我也是配置完项目启动失败才发现这个issue，懒得切换版本就暂且不使用starter pom吧，直接使用编程式配置。

###### **准备**

本Demo中使用的两个数据源是db0和db1，每个数据源之中包含了两组表t_order_0和t_order_1，t_order_item_0和t_order_item_1 。和官方文档的demo一致，这两组表的建表语句为：

```sql
CREATE TABLE IF NOT EXISTS t_order_x (
  order_id INT NOT NULL,
  user_id  INT NOT NULL,
  PRIMARY KEY (order_id)
);
CREATE TABLE IF NOT EXISTS t_order_item_x (
  item_id  INT NOT NULL,
  order_id INT NOT NULL,
  user_id  INT NOT NULL,
  PRIMARY KEY (item_id)
);
```

逻辑结构如下：

```sh
db0
  ├── t_order_0 
  └── t_order_1 
db1
  ├── t_order_0 
  └── t_order_1
```

首先引入依赖

```
<dependency>
       <groupId>io.shardingjdbc</groupId>
       <artifactId>sharding-jdbc-core</artifactId>
       <version>2.0.3</version>
</dependency>
```

配置表分片策略

```java
 @Bean
    TableRuleConfiguration getOrderTableRuleConfiguration() {
        TableRuleConfiguration orderTableRuleConfig = new TableRuleConfiguration();
        //配置逻辑表名，并非数据库中真实存在的表名，而是sql中使用的那个，不受分片策略而改变.  
        //例如：select * frpm t_order where user_id = xxx
        orderTableRuleConfig.setLogicTable("t_order");
        //配置真实的数据节点，即数据库中真实存在的节点，由数据源名 + 表名组成
        //${} 是一个groovy表达式，[]表示枚举，{...}表示一个范围。  
        //整个inline表达式最终会是一个笛卡尔积，表示ds_0.t_order_0. ds_0.t_order_1
        // ds_1.t_order_0. ds_1.t_order_0
        orderTableRuleConfig.setActualDataNodes("ds_${0..1}.t_order_${0..1}");
        //主键生成列，默认的主键生成算法是snowflake
        orderTableRuleConfig.setKeyGeneratorColumnName("order_id");
       //设置分片策略，这里简单起见直接取模，也可以使用自定义算法来实现分片规则
        orderTableRuleConfig.setTableShardingStrategyConfig(new InlineShardingStrategyConfiguration("order_id","t_order_${order_id % 2}"));
        return orderTableRuleConfig;
    }

    @Bean
    TableRuleConfiguration getOrderItemTableRuleConfiguration() {
        TableRuleConfiguration orderItemTableRuleConfig = new TableRuleConfiguration();
        orderItemTableRuleConfig.setLogicTable("t_order_item");
        orderItemTableRuleConfig.setActualDataNodes("ds_${0..1}.t_order_item_${0..1}");
        orderItemTableRuleConfig.setTableShardingStrategyConfig(new InlineShardingStrategyConfiguration("order_item_id","t_order_item_${order_id % 2}"));

        return orderItemTableRuleConfig;
    }
```

配置数据源

```java
  private Map<String, DataSource> createDataSourceMap() {
        Map<String, DataSource> result = new HashMap<>(2, 1);
        result.put("ds_0", createDataSource("ds_0"));
        result.put("ds_1", createDataSource("ds_1"));
        return result;
    }

    private DataSource createDataSource(final String dataSourceName) {
        DruidDataSource result = new DruidDataSource();
        result.setInitialSize(10);
        result.setMinIdle(10);
        result.setMaxActive(50);
        result.setDriverClassName(com.mysql.jdbc.Driver.class.getName());
        result.setUrl(String.format("jdbc:mysql://localhost:3306/%s?useSSL=false", dataSourceName));
        result.setUsername("root");
        result.setPassword("");
        return result;
    }

   @Bean
    DataSource getShardingDataSource() throws SQLException {
        ShardingRuleConfiguration shardingRuleConfig = new ShardingRuleConfiguration();
        shardingRuleConfig.getTableRuleConfigs().add(getOrderTableRuleConfiguration());
        shardingRuleConfig.getTableRuleConfigs().add(getOrderItemTableRuleConfiguration());
        shardingRuleConfig.getBindingTableGroups().add("t_order, t_order_item");
        shardingRuleConfig.setDefaultDatabaseShardingStrategyConfig(  
        new InlineShardingStrategyConfiguration("user_id", "ds_${user_id % 2}"));
        shardingRuleConfig.setDefaultTableShardingStrategyConfig(  
        new InlineShardingStrategyConfiguration("order_id", "t_order_${order_id % 2}"));
        return ShardingDataSourceFactory.createDataSource(createDataSourceMap(),  
        shardingRuleConfig, new HashMap<>(), null);
    }
```

###### **结果**

使用junittest插入一条记录，查看分片结果：

```java
@SpringBootTest
@RunWith(SpringRunner.class)
public class OrderDaoTest {


    @Autowired private OrderDao orderDao;

    @Test
    public void addOrder() {
        Order order = new Order();
        order.setUserId(1);
        order.setOrderId(1);
        //insert into t_order (order_id, user_id) values(#{orderId}, #{userId})
        orderDao.addOrder(order);
    }
}
```

t_order这张表配置的分片策略是按照order_id与2取模，分库策略则是按照user_id与2取模，
所以最终的结果应该是插入在ds_1中的t_order_1中。

### **5、存储过程**

#### **介绍：**

就是一组SQL语句集，功能强大，可以实现一些比较复杂的逻辑功能，类似于JAVA语言中的方法；

#### **优点**：

①、使用了存过程，很多相似性的删除，更新，新增等操作就变得轻松了，并且以后也便于管理！
②、存储过程因为SQL语句已经预编绎过了，因此运行的速度比较快。 
③、存储过程可以接受参数、输出参数、返回单个或多个结果集以及返回值。可以向程序返回错误原因。 
④、存储过程运行比较稳定，不会有太多的错误。只要一次成功，以后都会按这个程序运行。 
⑤、存储过程主要是在服务器上运行，减少对客户机的压力。 
⑥、存储过程可以包含程序流、逻辑以及对数据库的查询。同时可以实体封装和隐藏了数据逻辑。 
⑦、存储过程可以在单个存储过程中执行一系列SQL语句。 
⑧、存储过程可以从自己的存储过程内引用其它存储过程，这可以简化一系列复杂语句。

### **6、配置MySQL最大连接数（my.ini配置）**

```sql
--查看最大连接数
show variables like '%max_connections%';
--修改最大连接数
set GLOBAL max_connections = 200;
```

### **8、随时清理碎片**

#### **产生的原因？**	

①、表的存储会出现碎片化，每当删除了一行内容，该段空间就会变为空白、被留空，而在一段时间内的大量删除操作，会使这种留空的空间变得比存储列表内容所使用的空间更大；

②、当执行插入操作时，MySQL会尝试使用空白空间，但如果某个空白空间一直没有被大小合适的数据占用，仍然无法将其彻底占用，就形成了碎片；

③、当MySQL对数据进行扫描时，它扫描的对象实际是列表的容量需求上限，也就是数据被写入的区域中处于峰值位置的部分；

#### **如何处理？**

```sql
--查看表的碎片大小
SHOW TABLE STATUS LIKE '表名';
--清除表的碎片（MyISAM）	
OPTIMIZE TABLE table_name；
--清除表的碎片（InnoDB）
 alter table 表名 engine=InnoDB
```

### **9、SQL语句优化**

#### **关键词的执行顺序**

**写的顺序：**select ... from... where.... group by... having... order by.. limit [offset,] 
**执行顺序：**from... where...group by... having.... select ... order by... limit

```sh
 下面具体分析一下查询处理的每一个阶段
 1 FROM: 对FROM的左边的表和右边的表计算笛卡尔积。产生虚表VT1
 2 ON: 对虚表VT1进行ON筛选，只有那些符合<join-condition>的行才会被记录在虚表VT2中。
 3 JOIN： 如果指定了OUTER JOIN（比如left join、 right join），那么保留表中未匹配的行就会作为外部行添加到			虚拟表VT2中，产生虚拟表VT3, rug from子句中包含两个以上的表的话，那么就会对上一个join连接产生的结		   果VT3和下一个表重复执行步骤1~3这三个步骤，一直到处理完所有的表为止。
 4 WHERE： 对虚拟表VT3进行WHERE条件过滤。只有符合<where-condition>的记录才会被插入到虚拟表VT4中。
 5 GROUP BY: 根据group by子句中的列，对VT4中的记录进行分组操作，产生VT5.
 6 CUBE | ROLLUP: 对表VT5进行cube或者rollup操作，产生表VT6.
 7 HAVING： 对虚拟表VT6应用having过滤，只有符合<having-condition>的记录才会被 插入到虚拟表VT7中。
 8 SELECT： 执行select操作，选择指定的列，插入到虚拟表VT8中。
 9 DISTINCT： 对VT8中的记录进行去重。产生虚拟表VT9.
 10 ORDER BY: 将虚拟表VT9中的记录按照<order_by_list>进行排序操作，产生虚拟表VT10.
 11 LIMIT：取出指定行的记录，产生虚拟表VT11, 并将结果返回。
```

**详细见SQL语句优化文档**

### 10、执行计划

#### **定位慢查询**

##### **什么是慢查询？**

mysql默认sql语句在10S内没有返回，则被认为是慢查询

慢查询都有日志记录

使用show status查看mysql服务器状态信息

##### **命令：**

查看数据库启动时间：show status like 'uptime';

显示数据可的查询、更新、添加、删除的次数：show status like 'com_select',……

显示数据库的连接数：show status like 'connections';

显示慢查询次数：show status like 'slow_queries';

默认情况mysql不会记录慢查询的日志

 步骤：停止mysql服务->进入mysql目录->进入cmd->输入 mysqld.exe --safe-mode --slow-query-log->不要关闭cmd界面

查看慢查询log日志：在mysql data目录中找log文件（可以在my.ini中找到目录位置）

#### **索引优化**

##### **①、何时使用聚集索引或非聚集索引？**

| 动作描述           | 使用聚集索引 | 使用非聚集索引 |
| ------------------ | ------------ | -------------- |
| 列经常被分组排序   | 使用         | 使用           |
| 返回某范围内的数据 | 使用         | 不使用         |
| 一个或极少不同值   | 不使用       | 不使用         |
| 小数目的不同值     | 使用         | 不使用         |
| 大数目的不同值     | 不使用       | 使用           |
| 频繁更新的列       | 不使用       | 使用           |
| 外键列             | 使用         | 使用           |
| 主键列             | 使用         | 使用           |
| 频繁修改索引列     | 不使用       | 使用           |

##### **②、 索引不会包含有NULL值的列**

只要列中包含有NULL值都将不会被包含在索引中，复合索引中只要有一列含有NULL值，那么这一列对于此复合索引就是无效的。所以我们在数据库设计时不要让字段的默认值为NULL。

##### **③、使用短索引**

对串列进行索引，如果可能应该指定一个前缀长度。例如，如果有一个CHAR(255)的列，如果在前10个或20个字符内，多数值是惟一的，那么就不要对整个列进行索引。短索引不仅可以提高查询速度而且可以节省磁盘空间和I/O操作。

##### **④、 索引列排序**

MySQL查询只使用一个索引，因此如果where子句中已经使用了索引的话，那么order by中的列是不会使用索引的。因此数据库默认排序可以符合要求的情况下不要使用排序操作；尽量不要包含多个列的排序，如果需要最好给这些列创建复合索引。

##### **⑤、 like语句操作**

一般情况下不鼓励使用like操作，如果非使用不可，如何使用也是一个问题。like “%aaa%” 不会使用索引而like “aaa%”可以使用索引。

##### **⑥、 不要在列上进行运算**

例如：select * from users where YEAR(adddate)<2007，将在每个行上进行运算，这将导致索引失效而进行全表扫描，因此我们可以改成：select * from users where adddate<’2007-01-01′。

### **11、硬件优化**

![](/images/MySQL优化/10.gif)

#### **从图上可以看到基本上每种设备都有两个指标：**

延时（响应时间）：表示硬件的突发处理能力；

带宽（吞吐量）：代表硬件持续处理能力。

#### **从上图可以看出，计算机系统硬件性能从高到代依次为：**

CPU——Cache(L1-L2-L3)——内存——SSD硬盘——网络——硬盘

由于SSD硬盘还处于快速发展阶段，所以本文的内容不涉及SSD相关应用系统。

#### **根据数据库知识，我们可以列出每种硬件主要的工作内容：**

CPU及内存：缓存数据访问、比较、排序、事务检测、SQL解析、函数或逻辑运算；

网络：结果数据传输、SQL请求、远程数据库访问（dblink）；

硬盘：数据访问、数据写入、日志记录、[大数据](http://lib.csdn.net/base/hadoop)量排序、大表连接。

#### **性能基本优化法则：**

![](/images/MySQL优化/11.gif)

**优化法则归纳为5个层次：**

1、  减少数据访问（减少磁盘访问）

2、  返回更少数据（减少网络传输或磁盘访问）

3、  减少交互次数（减少网络传输）

4、  减少服务器CPU开销（减少CPU及内存开销）

5、  利用更多资源（增加资源）

 由于每一层优化法则都是解决其对应硬件的性能问题，所以带来的性能提升比例也不一样。传统数据库系统设计是也是尽可能对低速设备提供优化方法，因此针对低速设备问题的可优化手段也更多，优化成本也更低。我们任何一个SQL的性能优化都应该按这个规则由上到下来诊断问题并提出解决方案，而不应该首先想到的是增加资源解决问题。

#### **以下是每个优化法则层级对应优化效果及成本经验参考：**

| **优化法则**      | **性能提升效果** | **优化成本** |
| ----------------- | ---------------- | ------------ |
| 减少数据访问      | 1~1000           | 低           |
| 返回更少数据      | 1~100            | 低           |
| 减少交互次数      | 1~20             | 低           |
| 减少服务器CPU开销 | 1~5              | 低           |
| 利用更多资源      | @~10             | 高           |

### **参考：**

分库/分表实现参考：[codingBoyJack](https://www.jianshu.com/u/138209947f2b)