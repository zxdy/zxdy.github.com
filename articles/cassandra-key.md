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

但是，问题又来了, 在查询时，cassandra查询的where语句中必须指定所有的partition key，这是最基本的条件。比如:
```SQL
PRIMARY KEY((key1，key2)，key3，key4，...)
```

* 合法的查询(不考虑二级索引)

where key1 and key2

where key1 and key2 and key3

where key1 and key2 and key3 and key4

* 非法的查询

where key1

where key2

where key1 and key2 and key4


所以parition key字段过多会对以后的查询造成很大困扰，在建表的时候首先一定要考虑好数据模型，以免后期掉坑。此外假如与spark集成的话，可以在一定程度上规避掉上面非法查询的问题，通过sparksql可以近似实现关系型数据库sql的查询，而不用考虑查询中一定要带上所有partition key字段。
