在任何一样计算机软件产品中，当我们需要考虑其性能的时候，往往都会从CPU，IO，Network这三个方面考虑。CPU代表着处理问题的能力，IO代表着存储的吞吐能力，Network代表着数据传输的能力。oracle当然也不例外。

下图反映的是一个应用程序总体的响应时间的分布情况。用户在前端发出数据的请求之后，经过网络层，到达数据库服务器。数据库服务器接收请求，然后对SQL进行语法语义分析，然后生成执行计划，接着执行sql，取得数据最后再次经过网络层返回到前端展现给用户。

![bottle-neck-of-oracle][1]

很显然 **oracle的处理时间=cpu处理时间+[资源等待时间+Disk IO 等待时间]**。

根据以上公式，我们可以发现只要能有效地利用cpu资源和降低等待时间就可以提高数据库的性能，使它处理得更快。

这篇文章主要关注oracle sql在cpu上的性能优化。个人感觉相比等待上的优化简单一些，等待很多时候涉及到并发，资源争用，数据块等知识，优化也更加复杂不好下手。

**那么问题来了：**
## 1. 首先，怎么查看CPU信息

CPU的多少在很大程度上（质量也是很重要滴）决定了数据库性能的好坏。越多的CPU，可以并发的数目就越多。

* 用系统命令查看。这里我默认认为oracle是安装在linux服务器上，当然装在windows上的不是没有，只是略奇葩了。

    refer to  [查看cpu信息][2]

* sql查询NUM_CPUS字段。v$osstat这张表包含了很多有用的信息
附上[官方文档][3]解释

```sql
SELECT * FROM v$osstat; 
```

* awr报告。在awr中会有cpu相关的各种报告，包括硬件信息以及更重要的性能信息。关于cpu的性能分析我会在后面展开。下面这张图展现了当前实例使用的cpu硬件信息。

![awr cpu][4]

>CPUs: 逻辑cpu数
Cores: cpu核数
Sockets: 物理cpu数

## 2. 怎么看CPU的性能

上面的CPU硬件信息只是帮助我们有个初步的概念，如果说你的数据库性能很差，CPU又很烂，很烂还没几个，那你真的该先换CPU了。。

换完CPU，我们先来了解下下面这两个概念：

* host cpu
* instance cpu

在AWR报告中会有这样两个不同的分类，像是这样

![host cpu][5]

还有这样

![instance cpu][6]

它们其实是分别代表了服务器cpu的负载和oracle实例的负载情况。

对于host cpu：

* "Load Average" begin/end值代表CPU的大致运行队列大小。上图中快照开始
到结束，平均 CPU负载减少了。
* %User+%System=> 总的CPU使用率，在这里是5.7%。
* Busy Time=Elapsed Time \* NUM_CPUS \* CPU utilization

对于instance cpu:

* %Total CPU,该实例所使用的CPU占总CPU的比例 -> % of total CPU for
Instance
* %Busy CPU，该实例所使用的Cpu占总的被使用CPU的比例 -> % of busy CPU for Instance。例如共4个逻辑CPU，其中3个被完全使用， 3个中的1 个完全被该实例使用，则%Total CPU= 1/4 =25%，而%Busy CPU= 1/3= 33%
* 当CPU高时一般看%Busy CPU可以确定CPU到底是否是本实例消耗的，还是
主机上其他程序。

身为一个开发我还要时不时看awr报告也是蛮拼的。一般来讲，awr更多的是展现数据库整体上的性能分析，你可能在以上的host cpu以及instance cpu上发现cpu的负载很高，但这又有什么用呢？是不是觉得没法继续了呢？当然不是，awr还提供了更多的详细的报告帮助我们定位哪些sql的cpu占用比较厉害：

1. AWR SQL ordered by Elapsed Time：

![AWR SQL ordered by Elapsed Time][7]

>%CPU - CPU Time as a percentage of Elapsed Time -> 这个语句耗费的DB TIME里CPU TIME占多少比例 -> 这个语句是否是CPU敏感的

2.AWR SQL ordered by CPU Time：

![AWR SQL order by cpu time][8]

> 第一列CPU TIME统计这个sql总共在快照时间内总共花费的cpu时间，在这个值比较高的情况下，如果相应的%CPU值也很高， 说明这个sql在cpu上的负载很高，需要考虑优化。

**如果发现有比较突出的sql，很可能就是瓶颈所在，可以继续跑个@?/rdbms/admin/awrsqrpt.sql看看**


  [1]: http://zxdy-blog.qiniudn.com/bottle-neck-of-oracle.png
  [2]: http://zxdy.github.io/articles/linux-info-check.html
  [3]: http://docs.oracle.com/cd/E11882_01/server.112/e40402/dynviews_2085.htm#REFRN30321
  [4]: http://zxdy-blog.qiniudn.com/awr_cpu.png
  [5]: http://zxdy-blog.qiniudn.com/awr_cpu.png
  [6]: http://zxdy-blog.qiniudn.com/instance_cpu.png
  [7]: http://zxdy-blog.qiniudn.com/sql_order_by_elapsed_time.png
  [8]: http://zxdy-blog.qiniudn.com/sql_order_by_cpu_time.png