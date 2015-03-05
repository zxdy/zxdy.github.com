通常情况下对oracle的表建立索引的时候并不需要考虑效率问题，这个通常情况指的是相应的表数据在百万级以下。但是一旦数据量大到千万级，亿级甚至更大的时候，我们就不能不考虑新建索引的效率问题，因为当表在建立索引的时候，会产生锁阻塞新数据的更新，如果索引不能很快地建立完毕，会对生产环境造成影响。

#1. PGA设置

hash_area_size： 这个参数控制每个会话的hash内存空间有多大。它也可以在会话级和实例级被修改。默认值是sort area空间大小的两倍
sort_area_size:  因为排序通常是在PGA中进行的，需要防止因空间或内存不足导致需要disk排序。

```sql
alter session set workarea_size_policy=manual;
alter session set hash_area_size=100000; 
alter session set sort_area_size=2000000000; -- 在系统可用内存足够的情况下，最大可以到2G
```

*question*
>1. 什么是PGA
2. 什么是SGA

#2. 增加temp表空间

因为大表的数据量比较大，导致建索引时需要的temp表空间也比较大，一般来说接近索引的大小，没把握的情况下可以估算一下索引的大小先：

```sql
set serveroutput on
declare
l_index_ddl varchar(1000);
l_used_bytes number
l_allocated_bytes number;
begin
dbms_space.create_index_cost(
ddl =>' create index ids_t on user(userid) ',
used_bytes=>l_used_bytes
alloc_bytes =>l_allocated_bytes);
dbms_ouput.put_line('used =' ||    'bytes' 
||'  allocated= ' || l_allocated_bytes || 'bytes');
end;
/
```

另外在建索引的过程中也可以随时监控表空间的使用情况，一旦发现temp表空间不够，可以随时加大

```sql
--查询表空间使用情况
SELECT UPPER(F.TABLESPACE_NAME) "表空间名",
D.TOT_GROOTTE_MB "表空间大小(M)",
D.TOT_GROOTTE_MB - F.TOTAL_BYTES "已使用空间(M)",
TO_CHAR(ROUND((D.TOT_GROOTTE_MB - F.TOTAL_BYTES) / D.TOT_GROOTTE_MB * 100,2),'990.99') "使用比",
F.TOTAL_BYTES "空闲空间(M)",
F.MAX_BYTES "最大块(M)"
FROM (SELECT TABLESPACE_NAME,
ROUND(SUM(BYTES) / (1024 * 1024), 2) TOTAL_BYTES,
ROUND(MAX(BYTES) / (1024 * 1024), 2) MAX_BYTES
FROM SYS.DBA_FREE_SPACE
GROUP BY TABLESPACE_NAME) F,
(SELECT DD.TABLESPACE_NAME,
ROUND(SUM(DD.BYTES) / (1024 * 1024), 2) TOT_GROOTTE_MB
FROM SYS.DBA_DATA_FILES DD
GROUP BY DD.TABLESPACE_NAME) D
WHERE D.TABLESPACE_NAME = F.TABLESPACE_NAME
ORDER BY 4 DESC;

select file_name,tablespace_name,autoextensible from dba_data_files

--增加表空间大小
alter tablespace USERS add datafile '/opt/ora9/users02.dbf' size 50M autoextend on next 50M maxsize UNLIMITED;
```

#3. 使用并行参数

关于利用并行度创建索引，前提是多个CPU，单CPU下用并行度创建索引，可能会造成资源的争用。理论上来说8个CPU, 可以用parallel 6 ,最多占用6个CPU,另外留下两个CPU供其他进程使用。
查看CPU核数的方法有很多，详细见（oracle性能优化-CPU篇）。最简单地就是用下面这个sql直接查

```sql
SELECT * FROM v$osstat where stat_name='NUM_CPUS'; 
```

#4. 使用nologging

nologging, 绝对应该使用，能减少大量redo log，使速度大幅上升。

于是一个比较标准的并行nologging建索引语句就出炉了。对于生产环境，保险的办法是再加上online参数，避免建索引时的锁对dml产生阻塞。

```sql
CREATE INDEX  table_idx ON table (col )  NOLOGGING PARALLEL 6;
```


*Note*
>对于一个比较大的操作，oracle可能会有等待事件发生
首先可以通过sql developer查看等待时间的信息，得到等待时间的p1，p2，p3。然后通过下面的sql转换p1，p2得到具体等待的object。

>```sql
select 
   owner,
   segment_name,
   segment_type
from 
   dba_extents
where 
   file_id = &P1 and &P2 between block_id and block_id + blocks -1;
```