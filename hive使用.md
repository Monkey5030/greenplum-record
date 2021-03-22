# Hive是基于Hadoop的数据仓库解决方案。由于Hadoop本身在数据存储和计算方面有很好的可扩展性和高容错性，因此使用Hive构建的数据仓库也秉承了这些特性。
## Impala与Hive  
[hive与数据库对比](https://github.com/Monkey5030/greenplum-record/blob/main/image/hive%E4%B8%8E%E4%BC%A0%E7%BB%9F%E7%9A%84%E6%AF%94%E8%BE%83.jpg)
* 不同点：  
Hive适合长时间批处理查询分析；而Impala适合进行交互式SQL查询。  
Hive依赖于MR计算框架，执行计划组合成管道型MR任务模型进行执行；而Impala则把执行计划表现为一棵完整的执行计划树，可更自然地分发执行计划到各个Impalad执行查询。  
Hive在执行过程中，若内存放不下所有数据，则会使用外存，以保证查询能够顺利执行完成；而Impala在遇到内存放不下数据时，不会利用外存，所以Impala处理查询时会受到一定的限制。  
* 相同点：  
使用相同的存储数据池，都支持把数据存储在HDFS和HBase中，其中HDFS支持存储TEXT、RCFILE、PARQUET、AVRO、ETC等格式的数据，HBase存储表中记录。  
使用相同的元数据。  
对SQL的解析处理比较类似，都是通过词法分析生成执行计划。 


## hive元数据  
[hive元数据学习](http://lxw1234.com/archives/2015/07/378.htm)  
* 查询表有多少数据  
```
select c.NAME, a.TBL_NAME, b.PARAM_KEY, b.PARAM_VALUE from metastore.TBLS as a left  join metastore.TABLE_PARAMS as b on a.TBL_ID = b.TBL_ID 
left  join metastore.DBS c on  a.DB_ID = c.DB_ID 
where   b.PARAM_KEY = 'numRows';
```
![image](https://github.com/Monkey5030/greenplum-record/blob/main/image/hive%E5%85%83%E6%95%B0%E6%8D%AE.png)  


## Hive SQL的优化  
1. 使用分区剪裁、列剪裁
在SELECT中，只拿需要的列，如果有，尽量使用分区过滤，少用SELECT *。  
在分区剪裁中，当使用外关联时，如果将副表的过滤条件写在Where后面，那么就会先全表关联，之后再过滤，比如：  
`SELECT a.id
FROM lxw1234_a a
left outer join t_lxw1234_partitioned b
ON (a.id = b.url);
WHERE b.day = ‘2015-05-10′`

    正确的写法是写在ON后面：  
    `
    SELECT a.id
    FROM lxw1234_a a
    left outer join t_lxw1234_partitioned b
    ON (a.id = b.url AND b.day = ‘2015-05-10′);
    `  
    或者直接写成子查询:  
    `SELECT a.id
    FROM lxw1234_a a
    left outer join (SELECT url FROM t_lxw1234_partitioned WHERE day = ‘2015-05-10′) b
    ON (a.id = b.url)`

2. 少用COUNT DISTINCT  
    数据量小的时候无所谓，数据量大的情况下，由于COUNT DISTINCT操作需要用一个Reduce Task来完成，这一个Reduce需要处理的数据量太大，就会导致整个Job很难完成，一般COUNT DISTINCT使用先GROUP BY再COUNT的方式替换：  

    `SELECT day,    COUNT(DISTINCT id) AS uv    FROM lxw1234    GROUP BY day`  
    可以转换成：  
    `SELECT day,
    COUNT(id) AS uv
    FROM (SELECT day,id FROM lxw1234 GROUP BY day,id) a
    GROUP BY day;`  
    虽然会多用一个Job来完成，但在数据量大的情况下，这个绝对是值得的。

3. 是否存在多对多的关联    
  只要遇到表关联，就必须得调研一下，是否存在多对多的关联，起码得保证有一个表或者结果集的关联键不重复。  
  如果某一个关联键的记录数非常多，那么分配到该Reduce Task中的数据量将非常大，导致整个Job很难完成，甚至根本跑不出来。  
  还有就是避免笛卡尔积，同理，如果某一个键的数据量非常大，也是很难完成Job的。  

4. 合理使用MapJoin  
  关于MapJoin的原理和机制，请参考 [一起学Hive]之十 。
  MapJoin中小表的大小可以用参数来调节。

5. 合理使用Union All  
  对同一张表的union all 要比multi insert快的多。
  具体请见：http://superlxw1234.iteye.com/blog/1536440

6. 并行执行Job  
  用过oracle rac的应该都知道parallel的用途。  
  并行执行的确可以大的加快任务的执行速率，但不会减少其占用的资源。  
  在hive中也有并行执行的选项。  
  具体请见：http://superlxw1234.iteye.com/blog/1703713  
  
7. 使用本地MR  
  如果在hive中运行的sql本身数据量很小，那么使用本地mr的效率要比提交到Hadoop集群中运行快很多。  
  具体请见：http://superlxw1234.iteye.com/blog/1703546  

8. 合理使用动态分区  
  参见 [一起学Hive]之六-Hive的动态分区  
  http://lxw1234.com/archives/2015/06/286.htm  
  
9. 避免数据倾斜  
  数据倾斜是Hive开发中对性能影响的一大杀手。  
  症状：任务迚度长时间维持在99%（或100%）;  
  查看任务监控页面，发现只有少量（1个或几个）reduce子任务未完成。  
  本地读写数据量很大。  
  导致数据倾斜的操作：  
  GROUP BY, COUNT DISTINCT, join  
  原因：key分布不均匀  
  业务数据本身特点  
  这里列出一些常用的数据倾斜解决办法：  
  使用COUNT DISTINCT和GROUP BY造成的数据倾斜：  
  存在大量空值或NULL，或者某一个值的记录特别多，可以先把该值过滤掉，在最后单独处理:  
  SELECT CAST(COUNT(DISTINCT imei)+1 AS bigint)  
  FROM lxw1234 where pt = ‘2012-05-28′  
  AND imei <> ‘lxw1234′ ;  
  比如某一天的IMEI值为’lxw1234’的特别多，当我要统计总的IMEI数，可以先统计不为’lxw1234’的，之后再加1.  
  多重COUNT DISTINCT  
  通常使用UNION ALL + ROW_NUMBER() + SUM + GROUP BY来变通实现。  
  使用JOIN引起的数据倾斜  
  关联键存在大量空值或者某一特殊值，如”NULL”  
  空值单独处理，不参与关联；  
  空值或特殊值加随机数作为关联键；  
  不同数据类型的字段关联  
  转换为同一数据类型之后再做关联  
10. 控制Map数和Reduce数  
  参见http://lxw1234.com/archives/2015/04/15.htm  
  
11. 中间结果压缩  
  参见 http://superlxw1234.iteye.com/blog/1741103  
  
12. 其他  
  在MapReduce的WEB界面上，关注Hive Job执行的情况；  
  了解HQL -> MapReduce的过程；  
  HQL优化其实也是MapReduce的优化，作为分布式计算模型，其最核心的地方就是要确保每个节点上分布的数据均匀，才能最大程度发挥它的威力，否则，某一个不均匀的节点就会拖后腿。  
