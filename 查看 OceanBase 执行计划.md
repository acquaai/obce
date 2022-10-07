## 实验内容

+ 掌握 OceanBase 的执行计划查看方法，包括 explain 命令和查看实际执行计划。


## 创建实验租户

```sh
obclient -h172.16.120.174 -uroot@sys#obce-single -P2883 -prootPWD123 -c -A oceanbase
```

```sql
CREATE RESOURCE UNIT 6c2g
       max_cpu 6,
       min_cpu 4,
       max_memory '2G',
       min_memory '1G',
       max_iops 1000,
       min_iops 800,
       max_disk_size '30G',
       max_session_num 300
     ;

CREATE RESOURCE POOL rp1
        UNIT '6c2g',
        UNIT_NUM = 1,
        ZONE_LIST = ('zone1')
      ;     


CREATE TENANT IF NOT EXISTS tpcc
      CHARSET='utf8mb4',
      ZONE_LIST=('zone1'), 
      PRIMARY_ZONE='RANDOM', 
      RESOURCE_POOL_LIST=('rp1') 
    SET ob_tcp_invited_nodes='%' 
    ;
```

**创建 benchmark 数据库，修改 tpcc 租户 root 密码**

```sh
obclient -h127.0.0.1 -uroot@tpcc -P2881 -c -A oceanbase

create database benchmark;
alter user 'root'@'%' identified by 'password';
```


## 配置 BenchmarkSQL

**下载 BenchmarkSQL**

```sh
[root@obdocker ~]# git clone https://github.com/obpilot/benchmarksql-5.0.git
```

**修改 props.ob 配置文件**

```sh
[root@obdocker ~]# cd benchmarksql-5.0/run/
[root@obdocker run]# cp props.ob{,bk}
[root@obdocker run]# vi props.ob

db=oracle
driver=com.alipay.oceanbase.obproxy.mysql.jdbc.Driver
conn=jdbc:oceanbase://127.0.0.1:2883/benchmark?useUnicode=true&characterEncoding=utf-8
user=root@tpcc#obce-singl
password=password
```

**修改 runSQL.sh, runLoader.sh, runBenchmark.sh 文件**

注释 `#source funcs.sh $1` 语句，根据 BenchmarkSQL 的安装路径修改为 `source /root/benchmarksql-5.0/run/funcs.sh $1`。

**创建表**

```sh
[root@obdocker run]# sh runSQL.sh props.ob sql.common/tableCreates.sql
```

**加载 Benchmark 数据**

```sh
[root@obdocker run]# sh runLoader.sh props.ob
Starting BenchmarkSQL LoadData

driver=com.alipay.oceanbase.obproxy.mysql.jdbc.Driver
conn=jdbc:oceanbase://127.0.0.1:2883/benchmark?useUnicode=true&characterEncoding=utf-8
user=root@tpcc#obce-single
password=***********
warehouses=2
loadWorkers=2
fileLocation (not defined)
csvNullValue (not defined - using default 'NULL')

Worker 000: Loading ITEM
Worker 001: Loading Warehouse      1
Worker 000: Loading ITEM done
Worker 000: Loading Warehouse      2
Worker 001: ERROR: Transaction is timeout
Worker 000: ERROR: Transaction is timeout
```

**事务执行超时，修改 tpcc 租户相关参数**

```sql
obclient -h127.0.0.1 -uroot@tpcc -P2883 -ppassword -c -A oceanbase

set global ob_query_timeout=36000000000;
set global ob_trx_timeout=36000000000;
set global max_allowed_packet=67108864;
set global ob_sql_work_area_percentage=100;
set global parallel_max_servers=800;
set global parallel_servers_target=800;
```

**再次加载**

```sh
[root@obdocker run]# sh runLoader.sh props.ob
Starting BenchmarkSQL LoadData

driver=com.alipay.oceanbase.obproxy.mysql.jdbc.Driver
conn=jdbc:oceanbase://127.0.0.1:2883/benchmark?useUnicode=true&characterEncoding=utf-8
user=root@tpcc#obce-single
password=***********
warehouses=2
loadWorkers=2
fileLocation (not defined)
csvNullValue (not defined - using default 'NULL')

Worker 000: Loading ITEM
Worker 001: Loading Warehouse      1
Worker 000: Loading ITEM done
Worker 000: Loading Warehouse      2
Worker 001: Loading Warehouse      1 done
Worker 000: Loading Warehouse      2 done
```

**创建表索引**

```sql
[root@obdocker ~]# obclient -h127.0.0.1 -uroot@tpcc -P2883 -ppassword -c -A benchmark

obclient [benchmark]> create index bmsql_customer_idx1 on bmsql_customer (c_w_id, c_d_id, c_last, c_first) local;
Query OK, 0 rows affected (0.567 sec)

obclient [benchmark]> create index bmsql_oorder_idx1 on bmsql_oorder (o_w_id, o_d_id, o_carrier_id, o_id) local;
Query OK, 0 rows affected (0.532 sec)
```


**执行性能测试**

```sh
[root@obdocker run]# sh runBenchmark.sh props.ob

.....
19:50:53,534 [main] INFO   jTPCC : Term-00,
Term-00, Running Average tpmTOTAL: 6.65    Current tpmTOTAL: 36    Memory Usage: 20MB / 232MB
19:51:56,685 [Thread-2] INFO   jTPCC : Term-00,
19:51:56,685 [Thread-2] INFO   jTPCC : Term-00,
19:51:56,685 [Thread-2] INFO   jTPCC : Term-00, Measured tpmC (NewOrders) = 4.75
19:51:56,685 [Thread-2] INFO   jTPCC : Term-00, Measured tpmTOTAL = 6.65
19:51:56,685 [Thread-2] INFO   jTPCC : Term-00, Session Start     = 2022-10-05 19:50:53
19:51:56,685 [Thread-2] INFO   jTPCC : Term-00, Session End       = 2022-10-05 19:51:56
19:51:56,686 [Thread-2] INFO   jTPCC : Term-00, Transaction Count = 6
[root@obdocker run]#
```


## TOP-SQL 分析

```sql
[root@obdocker ~]# obclient -h127.0.0.1 -uroot@tpcc -P2883 -ppassword -c -A oceanbase

obclient [oceanbase]> SELECT sql_id, count(*), round(avg(elapsed_time)) avg_elapsed_time,
    ->                      round(avg(execute_time)) avg_exec_time,
    ->                      s.svr_ip,
    ->                      s.svr_port,
    ->                      s.tenant_id,
    ->                      s.plan_id
    ->                     FROM gv$sql_audit s
    ->                     WHERE 1=1
    ->                      and request_time >= time_to_usec(DATE_SUB(current_timestamp, INTERVAL 30 MINUTE) )
    ->                     GROUP BY sql_id
    ->                    order by avg_elapsed_time desc limit 3;
+----------------------------------+----------+------------------+---------------+-----------+----------+-----------+---------+
| sql_id                           | count(*) | avg_elapsed_time | avg_exec_time | svr_ip    | svr_port | tenant_id | plan_id |
+----------------------------------+----------+------------------+---------------+-----------+----------+-----------+---------+
| 5GU86XHq9dc0czXpl6aLivGre7loN3hV |        1 |          1872446 |       1872334 | 127.0.0.1 |     2882 |      1002 |      74 |
| eLmpkIaniEuyojrkwcxuRBePe5XP   |        1 |          1737482 |       1737372 | 127.0.0.1 |     2882 |      1002 |      74 |
| BvKHdhnoSBZ1E��1�/ hZ83osK     |        1 |          1688853 |       1688749 | 127.0.0.1 |     2882 |      1002 |      74 |
+----------------------------------+----------+------------------+---------------+-----------+----------+-----------+---------+
3 rows in set (0.002 sec)
```


```sql
obclient [oceanbase]> select distinct query_sql from gv$sql_audit where sql_id='5GU86XHq9dc0czXpl6aLivGre7loN3hV';
+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| query_sql                                                                                                                                                                                                                                |
+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| SELECT s_quantity, s_data,        s_dist_01, s_dist_02, s_dist_03, s_dist_04,        s_dist_05, s_dist_06, s_dist_07, s_dist_08,        s_dist_09, s_dist_10     FROM bmsql_stock     WHERE s_w_id = 1 AND s_i_id = 92536     FOR UPDATE |
+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.005 sec)
```


**分析执行计划**

```sql
[root@obdocker ~]# obclient -h127.0.0.1 -uroot@tpcc -P2883 -ppassword -c -A benchmark

obclient [benchmark]> explain SELECT s_quantity, s_data,        s_dist_01, s_dist_02, s_dist_03, s_dist_04,        s_dist_05, s_dist_06, s_dist_07, s_dist_08,        s_dist_09, s_dist_10     FROM bmsql_stock     WHERE s_w_id = 1 AND s_i_id = 92536     FOR UPDATE \G
*************************** 1. row ***************************
Query Plan: ============================================
|ID|OPERATOR  |NAME       |EST. ROWS|COST  |
--------------------------------------------
|0 |TABLE SCAN|bmsql_stock|10       |124026|
============================================

Outputs & filters:
-------------------------------------
  0 - output([bmsql_stock.s_quantity], [bmsql_stock.s_data], [bmsql_stock.s_dist_01], [bmsql_stock.s_dist_02], [bmsql_stock.s_dist_03], [bmsql_stock.s_dist_04], [bmsql_stock.s_dist_05], [bmsql_stock.s_dist_06], [bmsql_stock.s_dist_07], [bmsql_stock.s_dist_08], [bmsql_stock.s_dist_09], [bmsql_stock.s_dist_10]), filter([bmsql_stock.s_w_id = 1], [bmsql_stock.s_i_id = 92536]),
      access([bmsql_stock.__pk_increment], [bmsql_stock.s_w_id], [bmsql_stock.s_i_id], [bmsql_stock.s_quantity], [bmsql_stock.s_data], [bmsql_stock.s_dist_01], [bmsql_stock.s_dist_02], [bmsql_stock.s_dist_03], [bmsql_stock.s_dist_04], [bmsql_stock.s_dist_05], [bmsql_stock.s_dist_06], [bmsql_stock.s_dist_07], [bmsql_stock.s_dist_08], [bmsql_stock.s_dist_09], [bmsql_stock.s_dist_10]), partitions(p0)

1 row in set (0.001 sec)
```


**Ref**

+ [OB Docs](https://www.oceanbase.com/docs/enterprise-oceanbase-database-cn-10000000000355150)
