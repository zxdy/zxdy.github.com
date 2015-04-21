上篇在讲到cassandra查询语句之坑的时候，有提到一个叫allow filtering的东东，但是令人费解的是，这货有时候出现有时候又不用出现。那到底这是怎么用的呢，让我们来用一个例子说明。首先还是沿用上篇的table2：

```SQL
create table table2 (
  key_part_one text,
  key_part_two int,
  data text,
  PRIMARY KEY(key_part_one, key_part_two)
);
```
此时我们有了一个主键（key_part_one, key_part_two），一个分区键:key_part_one，一个clustering key：key_part_two。


如果你执行下面一个查询：

```sql
SELECT * FROM table2;
```
Cassandra会返回所有table2表中的数据（假如table2表的数据量非常大，这种查询会导致rpc timeout，可以在Cassandra配置文件中适当加大timeout的时间来解决这一问题，但是个人感觉治标不治本。目前发现用spark+Cassandra的方式可以比较好的处理大数据量的存储以及分析，有时间我会写一篇spark+Cassandra的集成攻略）。

按照关系型数据库的习惯，你可能还会做如下的查询：

```sql
SELECT * FROM table2 WHERE key_part_two = 1111;
```

然后cassandra就提示你要不要allow filtering一下：

>Bad Request: Cannot execute this query as it might involve data filtering and thus may have unpredictable performance. If you want to execute this query despite the performance unpredictability, use ALLOW FILTERING.

这是因为Cassandra发现它可能不能很效率地执行这个查询，所以做出了警告："Be careful. Executing this query as such might not be a good idea as it can use a lot of your computing resources".

跟所有的nosql数据库一样，cassandra存储数据的方式是key-value，这决定了它查询数据的模式是根据key一层层往下找数据。cassandra执行这个查询的唯一方式是取出所有的行，然后过滤得到key_part_two = 1111的数据。

类似于oracle的统计信息直方图倾斜，key_part_two的值分布可能会有两种极端的情况，一种是绝大多数的key_part_two都等于1111。另一种是绝大多数的key_part_two都不等于1111。第一种情况比较适用allow filtering，因为取出的所有行基本已经是最后的需要的结果集，效率还算可以。但是第二种情况因为取出的所有行要过滤掉绝大多数的数据才是最终结果集，所以用allow filtering的性能会非常低下，经常对这个字段查询的话还是对它增加二级索引会更加好一点。

cassandra提示allow filtering的本质是它认为当前的查询可能会有很大的性能问题，让你决定是不是强制执行，这就是他的意义。因此当你的CQL语句被cassandra拒绝执行的时候，你需要考虑你的数据模型以及你的目的去做出最优的选择，比如说改变数据模型，增加二级索引或者换一张表查询，而不是立刻就不假思索的根据提示加上allow filtering。

参考文章：
[A series on Cassandra – Part 2: Indexes and keys][1]

  [1]: http://blog.websudos.com/2014/08/a-series-on-cassandra-part-2-indexes-and-keys/