简单记录一下在使用cassandra与spark集成时踩到的坑

1. cassandra建表时注意全部使用小写，否则在使用cassandrasql时会抛错（这不科学。。）
    >Exception in thread "main" java.util.NoSuchElementException: key not found:

2. 尽量不要使用decimal，否则会抛出错误。而使用int或者是double，基本已经够用。 
    >Java.lang.ClassCastException: java.math.BigDecimal cannot be cast to org.apache.spark.sql.types.Decimal

3. 不能一次使用多表关联，仅支持两表关联。（在sparksql的测试用例中发现三表关联的例子，但是我在实际使用中始终会报错，为了不引起麻烦还是将多表关联拆分成两两关联进行处理）

4. 条件查询时，谓词列值为null的行不会被查询，必须要加上 is null或is not null

5. 不支持nvl，decode，但是可以用case when 或者if代替。

    ```SQL
    cc.cassandraSql("select case when d_value is NULL then 1 else d_value end as d_value from test.stackoverflow")
    ```

6. 不支持update。考虑新建dataframe代替。

7. spark读取cassandra某个表的并行度由"spark.cassandra.input.split.size"决定,这个参数会动态计算最后的分区数,和spark worker节点数无关。但是SparkContext可以被多个线程使用，这意味着同个Spark Application中的Job可以同时提交到Spark Cluster中，所以可以并行读取不同的表减少整体的等待时间，前提是spark有空闲的资源。

8. 不支持类似以下的查询

    ```SQL
    cc.cassandraSql("select a.* from test.stackoverflow as a")
    cc.cassandraSql("select * from test.stackoverflow as a where k_part_two in(select value from test.stackoverflow)")
    ```
9.  sparksql 语句中对子查询和表名尽量设置别名。


[sparksql 关键字][1]


  [1]: https://github.com/apache/spark/blob/master/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/SqlParser.scala