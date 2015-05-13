在spark的实际使用中，生成的结果集可能还会做为其他流程的数据源，进行再次分析处理，本文主要提供一个将spark结果集持久化到关系型数据库（postgresql）中的思路，毕竟对于小规模的数据，使用关系型数据库的sql处理展现数据比起nosql数据库会更加方便。

以spark自带的people.json为例，首先用它生成一个dataframe

```JAVA
//people.json
{"name":"Michael"}
{"name":"Andy", "age":30}
{"name":"Justin", "age":19}
//生成dataframe
val testdata=sqlContext.jsonFile("people.json")
```

现在我们有了一个类型为dataframe的testdata，假设这个就是处理到最后的结果集，那么如何将它保存到数据库呢？一个比较简单的想法是轮询testdata的所有rows，然后将每一个row中的值插入到数据库。

但是带来一个问题是，假如结果集很大，那这个单线程方案的效率就比较值得商榷了，最后的瓶颈很可能是出现在最后的数据库插入上。所以要想办法将其并行。

于是我想到spark的数据处理本身就是并行的，最后的结果集是按照分区分开生成，所以我们可以不用等最后完整的结果集生成之后再去存数据库，而是可以在每一个分区数据生成时就插数据，从而实现了存数据库的并行。 

但是在此之前，我们还需要考虑并行度的问题。因为数据库的连接数是有限制的，而RDD的最后分区数可能很大，假如一个分区就有一个数据库连接的话，最后会导致连接数不足而插入失败。所以在插入数据库之前，我们首先要修改RDD的分区数，保证它在一个合理的范围内，一般来说50个左右。


关于如何修改分区个数，在前一篇[spark job物理执行图实战][1]中，已经提到过修改分区个数的方法**repartition**，在此就不做过多介绍了。

下面是个简单的demo。要注意几点

1. demo中首先将testdata的分区数修改为4，然后调用mapPartitions对每个分区执行saveToPostgres方法，所以最终会有4个数据库连接。
2. 为什么有个collect()尾巴:因为mapPartitions只是个Transformation，而spark application job的触发是需要通过action来完成，光有mapPartitions还不够。

3. 这个demo比较简陋，没有考虑异常处理情况，在本文所用的数据源people.json中，有一行没有age，所以会抛空指针错误，要想正常运行的话，需要在people.json中补上age值。
4. 记得先在postgresql上新建一个对应的表
5. 别忘了jdbc包

**代码：**

```java
def main(args: Array[String]) {
    ....
    val testdata=sqlContext.jsonFile("people.json")
    testdata.repartition(4).mapPartitions(saveToPostgres).collect()
}

private def saveToPostgres(rows:Iterator[Row]): Iterator[Row] ={
    val url = "jdbc:postgresql://localhost/postgres"
    val user = "postgres"
    val password = "1234"
    val con = DriverManager.getConnection(url, user, password)
    val sqlstr = "INSERT INTO public.people(age,name) VALUES(?,?)"
    val stmt = con.prepareStatement(sqlstr)
    rows.foreach(insertRow(_,stmt))
    rows
  }

private def insertRow(row:Row,stmt:PreparedStatement): Unit ={
    var age=row.get(0).toString.toInt
    var name=row.get(1).toString
    stmt.setInt(1,age)
    stmt.setString(2,name)
    stmt.executeUpdate()
}
```

最后

还有没有更好的方法？从效率上讲，导数据到数据库最快的应该是通过批量load的方法（postgresql的copy api），像demo中的方式虽然做到了并行，但是本质还是一条条记录插入，速度肯定不如批量从文本导入。spark的saveAsTextFile可以用于生成文本文件，但它是一个action方法，所以要等到文本生成完才能导数据，这样一来的话，之后的事情其实跟spark已经关系不大了。只要新写个程序，将spark生成的文本拆分然后并行load即可。个人感觉，还是推荐这种方式，耦合性低，扩展性好，效率也高，而且对于异常数据的处理也都可以交给jdbc完成。


  [1]: http://zxdy.github.io/articles/spark-job-real-play.html