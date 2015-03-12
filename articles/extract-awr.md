# 1．AWR的启用

在默认情况下，Oracle启用数据库统计收集这项功能（即启用AWR）。是否启用AWR由初始化参数STATISTICS_LEVEL控制

```sql
SQL> SHOW PARAMETER STATISTICS_LEVEL 
NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
statistics_level                     string      TYPICAL
```

如果STATISTICS_LEVEL的值为TYPICAL或者 ALL，表示启用AWR；如果STATISTICS_LEVEL的值为BASIC，表示禁用AWR。

初始化参数statistics_level介绍：

AWR的行为受到参数STATISTICS_LEVEL的影响。这个参数有三个值：

* BASIC：awr统计的计算和衍生值关闭.只收集少量的数据库统计信息.

* TYPICAL：默认值．只有部分的统计收集.他们代表需要的典型监控oracle数据库的行为.

* ALL : 所有可能的统计都被捕捉. 并且有操作系统的一些信息.这个级别的捕捉应该在很少的情况下,比如你要更多的sql诊断信息的时候才使用.

# 2. 导出AWR报告

* AWR比对报告

```sql
@$ORACLE_HOME/rdbms/admin/awrddrpt.sql 
```

* RAC全局AWR

```sql
@$ORACLE_HOME/rdbms/admin/awrgrpt.sql
```

* 本实例的AWR报告，运行脚本awrrpt.sql。

```sql
@$ORACLE_HOME/rdbms/admin/awrrpt.sql 
```

* 产生某个实例（RAC）的AWR报告，运行脚本awrrpti.sql。

```sql
@$ORACLE_HOME/rdbms/admin/awrrpti.sql 
```

* 产生某条SQL语句的AWR报告，运行脚本awrsqrpt.sql。

```sql
@$ORACLE_HOME/rdbms/admin/awrsqrpt.sql 
```