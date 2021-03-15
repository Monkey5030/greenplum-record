
1. 数据库启动：gpstart  
常用可选参数： -a : 直接启动，不提示终端用户输入确认  
              -m:只启动master 实例，主要在故障处理时使用  

2.  数据库停止：gpstop：  

    常用可选参数：-a：直接停止，不提示终端用户输入确认  
                -m：只停止master 实例，与gpstart –m 对应使用  
                -M fast：停止数据库，中断所有数据库连接，回滚正在运行的事务  
                -u：不停止数据库，只加载pg_hba.conf 和postgresql.conf中运行时参数，当改动参数配置时候使用。  
                评：-a用在shell里，最多用的还是-M fast。

3. 查看实例配置和状态
                select * from gp_configuration order by 1 ;
    主要字段说明：
               Content：该字段相等的两个实例，是一对Ｐ（primary instance）和Ｍ（mirror  Instance)
               Isprimary：实例是否作为primary instance 运行
               Valid：实例是否有效，如处于false 状态，则说明该实例已经down 掉。
               Port：实例运行的端口
               Datadir:实例对应的数据目录

4. gpstate ：显示Greenplum数据库运行状态，详细配置等信息  

    常用可选参数：-c：primary instance 和 mirror instance 的对应关系  
                 -m：只列出mirror 实例的状态和配置信息  
                 -f：显示standby master 的详细信息  
                 -Q：显示状态综合信息  

    该命令默认列出数据库运行状态汇总信息，常用于日常巡检。
    评：最开始由于网卡驱动的问题，做了mirror后，segment经常down掉，用-Q参数查询综合信息还是比较有用的。

5.  查看用户会话和提交的查询等信息  

    select * from pg_stat_activity  该表能查看到当前数据库连接的IP 地址，用户名，提交的查询等。另外也可以在master 主机上查看进程，对每个客户端连接，master 都会创建一个进程。ps -ef |grep -i postgres |grep -i con

    评：常用的命令，我经常用这个查看数据库死在那个sql上了。

6. 查看数据库、表占用空间
    select pg_size_pretty(pg_relation_size('schema.tablename'));
    select pg_size_pretty(pg_database_size('databasename));
    必须在数据库所对应的存储系统里，至少保留30%的自由空间，日常巡检，要检查存储空间的剩余容量。
    评：可以查看任何数据库对象的占用空间，pg_size_pretty可以显示如mb之类的易读数据，另外，可以与pg_tables，pg_indexes之类的系统表链接，统计出各类关于数据库对象的空间信息。
7. 收集统计信息，回收空间
    定期使用Vacuum analyze tablename 回收垃圾和收集统计信息，尤其在大数据量删除，导入以后，非常重要
    评：这个说的不全面，vacuum分两种，一种是analize，优化查询计划的，还有一种是清理垃圾数据，postres删除工作，并不是真正删除数据，而是在被删除的数据上，坐一个标记，只有执行vacuum时，才会真正的物理删除，这个非常重用，有些经常更新的表，各种查询、更新效率会越来越慢，这个多是因为没有做vacuum的原因
    8. 查看数据分布情况

    两种方式：
      *  Select gp_segment_id,count(*) from  tablename  group by 1 ;
      *  在命令运行：gpskew -t public.ate -a postgres
    如数据分布不均匀，将发挥不了并行计算的优势，严重影响性能。

    评：非常有用，gp要保障数据分布均匀。
9. 实例恢复：gprecoverseg

    通过gpstate 或gp_configuration 发现有实例down 掉以后，使用该命令进行回复。
10.   查看锁信息：
      SELECT locktype, database, c.relname, l.relation, l.transactionid, l.transaction, l.pid, l.mode, l.granted, a.current_query
      FROM pg_locks l, pg_class c, pg_stat_activity a
      WHERE l.relation=c.oid AND l.pid=a.procpid
      ORDER BY c.relname;
    主要字段说明：
    relname: 表名
    locktype、mode 标识了锁的类型
    11. explain：在提交大的查询之前，使用explain分析执行计划、发现潜在优化机会，避免将系统资源熬尽。  
    评：少写了个analyze，如果只是explain，统计出来的执行时间，是非常坑爹的，如果希望获得准确的执行时间，必须加上analyze。

12.   数据库备份 gp_dump

      常用参数：-s: 只导出对象定义（表结构，函数等）
               -n: 只导出某个schema
      gp_dump 默认在master 的data 目录上产生这些文件：
      gp_catalog_1_<dbid>_<timestamp> ：关于数据库系统配置的备份文件
      gp_cdatabase_1_<dbid>_<timestamp>：数据库创建语句的备份文件
      gp_dump_1_<dbid>_<timestamp>：数据库对象ddl语句
      gp_dump_status_1_<dbid>_<timestamp>：备份操作的日志
      在每个segment instance 上的data目录上产生的文件：
      gp_dump_0_<dbid>_<timestamp>：用户数据备份文件
      gp_dump_status_0_<dbid>_<timestamp>：备份日志

13 .  数据库恢复 gp_restore  
        必选参数：--gp-k=key ：key 为gp_dump 导出来的文件的后缀时间戳
                 -d dbname  ：将备份文件恢复到dbname
14. 登陆与退出Greenplum  

    psql gpdb  
    psql -d gpdb -h gphostm -p 5432 -U gpadmin  
    使用utility方式  
    PGOPTIONS="-c gp_session_role=utility" psql -h -d dbname hostname -p port
    退出
    在psql命令行执行\q

15. 参数查询  

psql -c 'SHOW ALL;' -d gpdb

gpconfig --show max_connections

评：这个有用，可以管道给grep。

创建数据库

createdb -h localhost -p 5432 dhdw

创建GP文件系统

# 文件系统名

gpfsdw

# 子节点，视segment数创建目录

mkdir -p /gpfsdw/seg1

mkdir -p /gpfsdw/seg2

chown -R gpadmin:gpadmin /gpfsdw

# 主节点

mkdir -p /gpfsdw/master

chown -R gpadmin:gpadmin /gpfsdw

gpfilespace -o gpfilespace_config

gpfilespace -c gpfilespace_config

创建GP表空间

psql gpdb

create tablespace TBS_DW_DATA filespace gpfsdw;

SET default_tablespace = TBS_DW_DATA;

删除GP数据库

gpdeletesystem -d /gpmaster/gpseg-1 -f

查看segment配置

select * from gp_segment_configuration;

文件系统
select * from pg_filespace_entry;

磁盘、数据库空间
SELECT * FROM gp_toolkit.gp_disk_free ORDER BY dfsegment;

SELECT * FROM gp_toolkit.gp_size_of_database ORDER BY sodddatname;

日志
SELECT * FROM gp_toolkit.__gp_log_master_ext;

SELECT * FROM gp_toolkit.__gp_log_segment_ext;

表数据分布
SELECT gp_segment_id, count(*) FROM <table_name> GROUP BY gp_segment_id;

表占用空间
SELECT relname as name, sotdsize/1024/1024 as size_MB, sotdtoastsize as toast, sotdadditionalsize as other

   FROM gp_toolkit.gp_size_of_table_disk as sotd, pg_class

 WHERE sotd.sotdoid = pg_class.oid ORDER BY relname;

索引占用空间
SELECT soisize/1024/1024 as size_MB, relname as indexname

 FROM pg_class, gp_toolkit.gp_size_of_index

 WHERE pg_class.oid = gp_size_of_index.soioid

  AND pg_class.relkind='i';

OBJECT的操作统计
SELECT schemaname as schema, objname as table, usename as role, actionname as action, subtype as type, statime as time

 FROM pg_stat_operations

 WHERE objname = '<name>';

锁
SELECT locktype, database, c.relname, l.relation, l.transactionid, l.transaction, l.pid, l.mode, l.granted, a.current_query

 FROM pg_locks l, pg_class c, pg_stat_activity a

 WHERE l.relation=c.oid

  AND l.pid=a.procpid

 ORDER BY c.relname;

队列
SELECT * FROM pg_resqueue_status;

gpfdist外部表
# 启动服务

gpfdist -d /share/txt -p 8081 –l /share/txt/gpfdist.log &

# 创建外部表，分隔符为’/t’

drop EXTERNAL TABLE TD_APP_LOG_BUYER;

CREATE EXTERNAL TABLE TD_APP_LOG_BUYER (

 IP         text,

 ACCESSTIME text,

 REQMETHOD  text,

 URL        text,

 STATUSCODE int,

 REF        text,

 name       text,

 VID        text)

LOCATION ('gpfdist://gphostm:8081/xxx.txt')

FORMAT 'TEXT' (DELIMITER E'/t'

FILL MISSING FIELDS) SEGMENT REJECT LIMIT 1 percent;

# 创建普通表

create table test select * from TD_APP_LOG_BUYER;

# 索引

# CREATE INDEX idx_test ON test USING bitmap (ip);

# 查询数据

select ip , count(*) from test group by ip order by count(*);

gpload
# 创建控制文件

# 加载数据

gpload -f my_load.yml

copy
COPY country FROM '/data/gpdb/country_data'

 WITH DELIMITER '|' LOG ERRORS INTO err_country

 SEGMENT REJECT LIMIT 10 ROWS;

gpfdist外部表
# 创建可写外部表

CREATE WRITABLE EXTERNAL TABLE unload_expenses

( LIKE expenses )

LOCATION ('gpfdist://etlhost-1:8081/expenses1.out',

'gpfdist://etlhost-2:8081/expenses2.out')

FORMAT 'TEXT' (DELIMITER ',')

DISTRIBUTED BY (exp_id);

# 写权限

GRANT INSERT ON writable_ext_table TO <name>;

# 写数据

INSERT INTO writable_ext_table SELECT * FROM regular_table;

copy
COPY (SELECT * FROM country WHERE country_name LIKE 'A%') TO '/home/gpadmin/a_list_countries.out';

执行sql文件
psql gpdbname –f yoursqlfile.sql

或者psql登陆后执行

\i yoursqlfile.sql

# 命令
1. 创建数据库 　　createdb test_db;  
2. 删除数据库 　　dropdb test_db;  
3. 创建模式 　　　create schema myschema;  
4. 删除模式 　　　drop schema myschema;  
5. 创建用户 　　　create user user_name with password '123456' ;  
6. 删除用户 　　　drop user user_name;  
7. 查看系统用户信息 　　select usename from pg_user;  
8. 查看版本信息 　　　　select version();  
9. 打开psql交互工具 　　psql name_db;      
10. 执行sql文件 　　　　mydb=> \i basics.sql \i 命令从指定的文件中读取命令。  
11. 批量将文本文件中内容导入到wether表 　　　　　　 copy weather from '/home/user/weather.txt';  
12. 查看搜索模式 　　　 show search_path;  
13. 设置搜索模式 　　　 set search_path to myschema,public;  
14. 创建表空间 　　　　 create tablespace spacename_tb location 'file_path';  
15. 显示默认表空间 　　 show default_tablespace;  
16. 设置默认表空间 　　 set default_tablespace=表空间名称;  
17. 指定用户登录 　　　  psql mtps -u  
18. 显示当前系统时间 　　select now() ;  
19. 配置plpgsql语言 　　 create language 'plpgsql' handler plpgsql_call_handler;  
20. 删除规则 　　 drop rule name on relation [ cascade | restrict ];  
21. 当前日期属于一年中第几周 　　 select extract(week from timestamp '2020-06-14');  
22. 查询表是否存在 　　 select * from pg_statio_user_tables where relname='test_tb';  
23. 导出表 　　  　　　　 ./pg_dump -p 端口号 -u 用户 -t 表名称 -f 备份文件位置 数据库 ;  
24. 整个数据库导出 　　 pg_dumpall -d -p 端口号 -h 服务器ip -u postgres(用户名) > /home/xiaop/all.bak  
25. 数据库备份恢复 　　 psql -h 192.168.0.48 -p 5433 -u postgres  
26. 数据库备份 　　　　 pg_dumpall -h 192.168.0.4 -p 5433 -u postgres >/databack/postgresql2020061401.dmp  
27. 当前日期函数 　　　 select current_date;  
28. 返回第十条开始的5条记录 　　 select * from tbname limit 5 offset 10;  
29. 查看数据库大小 　　 select pg_size_pretty(pg_database_size('mtps')) as fulldbsize;  
30. 查看数据库表大小 　 select pg_size_pretty(pg_total_relation_size('test_db.t_l_collectfile')) as fulltblsize,pg_size_pretty(pg_relation_size('test_db.t_l_collectfile')) as jus      tthetblsize;  
31. 设置执行超过指定秒数的sql语句输出到日志 　　 log_min_duration_statement = 3  
32. 超过一定秒数sql自动执行执行计划　　　shared_preload_libraries = 'auto_explain',custom_variable_classes = 'auto_explain',auto_explain.log_min_duration = 4s  
33. 数据库备份  
　 　select pg_start_backup('backup baseline');  
　 　select pg_stop_backup();  
　 　recovery.conf  
　 　restore_command='cp /opt/buxlog/%f %p'  
34. 数据字典查看表结构 　　select column_name, data_type from information_schema.columns where table_name = 'test_tb';  
35. 查询表结构 　　　　  　 select a.attnum,a.attname as field,t.typname as type,a.attlen as length,a.atttypmod as lengthvar,a.attnotnull as notnull from pg_class c,pg_attribute a,pg_ type t where c.relname=表名称and a.attnum > 0 and a.attrelid = c.oid and a.atttypid = t.oid  
36. 将查询结果直接输出到文件，在psql中 \o 文件路径  
　 　select datname,rolname from pg_database a left outer join pg_roles b on a.datdba=b.oid; \o  
37. 查询数据库所有则 　　 select datname,rolname from pg_database a left outer join pg_roles b on a.datdba=b.oid ;  
38. 结束正在执行的事务 　　select * from pg_stat_activity;  
39. 查看被锁定表  
    select pg_class.relname as table, pg_database.datname as database, pid, mode, granted from pg_locks, pg_class, pg_database where pg_locks.relation = pg_class.oid  and            pg_locks.dat abase = pg_database.oid;  
40. 查看客户端连接情况 　　　　select client_addr ,client_port,waiting,query_start,current_query from pg_stat_activity;  
41. 常看数据库.conf配置 　　　  show all;  
　　修改数据库postgresql.conf参数  
　　修改postgresql.conf内容 pg_ctl reload  
　　回滚日志强制恢复 pg_resetxlog -f 数据库文件路径  
