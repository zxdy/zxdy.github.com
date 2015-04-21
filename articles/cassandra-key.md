在Cassandra中，primary key是一个非常重要的概念。关系型数据库中表可以没有primary key（主键），但是Cassandra中建表时必须指定primary key，它不仅决定了表的结构，而且还对数据查询方式的差异有巨大影响。partition key, composite key 和 clustering key共同组成了Cassandra的primary key。

为了说明它们的不同，我们先来看三张表。

```sql
create table table1 (
pri_key text PRIMARY KEY,
data text
);

create table table2 (
  key_part_one text,
  key_part_two int,
  data text,
  PRIMARY KEY(key_part_one, key_part_two)
);

create table table3 (
  key_part_one text,
  key_part_two int,
  key_clust_one text,
  key_clust_two int,
  key_clust_three uuid,
  data text,
  PRIMARY KEY((key_part_one,key_part_two), key_clust_one, key_clust_two, key_clust_three)
);

```
##格式区别

**table1**是初级版，primary key最简单的定义方式就是这样。此时pri_key就是partition key，clustering key。

**table2**是table1的升级版，格式为（key1，key2，key3，...）。key_part_one, key_part_two共同组成了primary key，这种方式即所谓的composite key（组合键）。其中key_part_one是partition key，key_part_two共同组成了primary是clustering key。

**table2**是高级版。这个primary key的组成最为复杂，格式为（（key1，key2），key3，key4，...）。其中key_part_one,key_part_two 共同组成了partition key。而外边的key_clust_one，key_clust_two, key_clust_three都是clustering key。

##实际的用途

Partition Key ：决定了数据在Cassandra各个节点的是如何分区的。

Clustering Key ： 用于在各个分区内的排序。

Primary Key ： 主键，决定数据行的唯一性

Composite Key ：只是一个多字段组合的概念

跟关系型数据库一样，分区都是为了解决大数据量查询的效率问题，所不同的是Cassandra的分区分布在各个节点上。注意同一个分区的数据是在同一个物理节点上的，这就造成一个问题，假如分区内的数据量过大的话，会造成Cassandra读取负载的不均衡，可以用类似于table3的建表方式，多个字段共同组成一个partition key减小单个分区的大小，使各个分区能够更均匀地分布在节点上，从而实现负载均衡。


## cassandra 查询之坑

在实际的使用过程中,cassandra的数据查询有很多不同于关系型数据库的地方，如果你总是用关系型数据库的思维去考虑cassandra的问题的话，往往会掉进坑里。cassandra的CQL写法并没有像你想象中的随心所欲，因为究其本质，它的数据集是以key-value的形式存放，所以在查询时会有很多限制。

让我们来看下上面三个表的查询语法会有哪些坑。

**table1** PRIMARY KEY(pri_key)

```SQL
select * from table1 where pri_key='111'; --good
select * from table1 where data='111'; --error
select * from table1 where pri_key='111' and data='111'; --error
```



**table2** PRIMARY KEY(key_part_one, key_part_two)

```SQL

select * from table2 where key_part_one='111'; --good
select * from table2 where key_part_two=111; --need allow filtering
select * from table2 where key_part_one='111' and key_part_two=111; --good
```

**table3** PRIMARY KEY((key_part_one,key_part_two), key_clust_one, key_clust_two, key_clust_three)

```SQL
select * from table3 where key_part_one='111'; --error
select * from table3 where key_part_two=111; --error
select * from table3 where key_part_one='111' and key_part_two=111; --good
select * from table3 where key_part_one='111' and key_part_two=111 and key_clust_one='111'; --good
select * from table3 where key_part_one='111' and key_part_two=111 and key_clust_two='111'; --error
select * from table3 where key_clust_one='111'; --need allow filtering
```

> 总结：
1. cassandra的查询必须在主键列上，或者查询的字段有二级索引。
2. 对于（A，B）形式的主键，假如查询条件不带分区键A，则查询语句需要开启allow filtering。
3. 对于（（A，B）,C,D）形式的主键，可以认为是第2点的变种。A，B必须同时出现在查询条件中,且C,D不可以跳跃，像where A and B and D的查询是非法的。
4. 以上查询不考虑范围查询的情况。


所以因为第三点的关系，parition key字段过多会对以后的查询造成很大困扰，在建表的时候首先一定要考虑好数据模型，以免后期掉坑。此外假如与spark集成的话，可以在一定程度上规避掉上面非法查询的问题，通过sparksql可以近似实现关系型数据库sql的查询，而不用考虑查询中一定要带上所有partition key字段。
