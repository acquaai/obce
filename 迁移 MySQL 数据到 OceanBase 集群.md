## 实验内容

+ 使用 mysqldump 将 MySQL 的表结构和数据同步到 OceanBase 的租户中
+ 使用 DataX 配置一个表的 MySQL 到 OceanBase 的租户的离线同步


## 准备数据

**安装 TPCC**

```sh
wget https://github.com/Percona-Lab/tpcc-mysql/archive/refs/heads/master.zip
cd tpcc-mysql-master/src && make
```

**MySQL 5.7 中创建 tpcc 数据库**

```sql
mysql> create database tpcc default character set utf8;
```

**创建欲加载数据的表结构**

```sh
[root@obdocker tpcc-mysql-master]# mysql -p -uroot tpcc < create_table.sql
```

```sql
mysql> use tpcc
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> show tables;
+----------------+
| Tables_in_tpcc |
+----------------+
| customer       |
| district       |
| history        |
| item           |
| new_orders     |
| order_line     |
| orders         |
| stock          |
| warehouse      |
+----------------+
9 rows in set (0.00 sec)
```

**将 TPCC 数据装载到 tpcc 数据库**

```sh
[root@obdocker tpcc-mysql-master]# ./tpcc_load -h 127.0.0.1 -d tpcc -u root -pFounder@123 -w 2
......
...DATA LOADING COMPLETED SUCCESSFULLY.
```

```sql
mysql> select count(*) from tpcc.customer;
+----------+
| count(*) |
+----------+
|    60000 |
+----------+
1 row in set (0.02 sec)
```


## 使用 mysqldump 迁移 tpcc 库的表到 OceanBase

**MySQL 导出表结构**

```sh
[root@obdocker ~]# mysqldump -h127.0.0.1 -uroot -pFounder@123 -d tpcc --compact > tpcc_ddl.sql
```

**MySQL 导出数据**

```sh
[root@obdocker ~]# mysqldump -h127.0.0.1 -uroot -pFounder@123 -t tpcc > tpcc_data.sql
```

**OceanBase 创建 tpcc 数据库**

```sh
obclient -h127.0.0.1 -uroot@sys#obce-single -P2883 -prootPWD123 -c -A oceanbase
obclient [oceanbase]> create database tpcc;
```

**将 MySQL 导出的表和数据导入 OceanBase 中**

```sql
[root@obdocker ~]# obclient -h127.0.0.1 -uroot@sys#obce-single -P2883 -prootPWD123 -c -A tpcc
obclient [tpcc]> source tpcc_ddl.sql
obclient [tpcc]> source tpcc_data.sql

obclient [tpcc]> show tables;
+----------------+
| Tables_in_tpcc |
+----------------+
| customer       |
| district       |
| history        |
| item           |
| new_orders     |
| order_line     |
| orders         |
| stock          |
| warehouse      |
+----------------+
9 rows in set (0.001 sec)

obclient [tpcc]> select count(*) from customer;
+----------+
| count(*) |
+----------+
|    60000 |
+----------+
1 row in set (0.139 sec)
```


## 使用 DataX 迁移 tpcc 库的表到 OceanBase

**安装 DataX**

```sh
[root@obdocker ~]# java -version
openjdk version "1.8.0_345"

[root@obdocker ~]# wget https://datax-opensource.oss-cn-hangzhou.aliyuncs.com/202209/datax.tar.gz
[root@obdocker ~]# tar xzf datax.tar.gz
[root@obdocker ~]# cd datax/bin
```


**编辑、修改配置文件(reader/writer)**

```json
[root@obdocker datax]# cat job/mysql2ob.json
{
    "job": {
        "setting": {
            "speed": {
                "channel": 4
            },
            "errorLimit": {
                "record": 0,
                "percentage": 0.1
            }
        },
        "content": [
            {
                "reader": {
                    "name": "mysqlreader",
                    "parameter": {
                        "username": "root",
                        "password": "Founder@123",
                        "column": [
                            "*"
                        ],
                        "connection": [
                            {
                                "table": [
                                    "warehouse"
                                ],
                                "jdbcUrl": ["jdbc:mysql://127.0.0.1:3306/tpcc?useUnicode=true&characterEncoding=utf8&useSSL=false"]
                            }
                        ]
                    }
                },

                "writer": {
                    "name": "oceanbasev10writer",
                    "parameter": {
                        "username": "root",
                        "password":"rootPWD123",
                      "obWriteMode": "insert",
                        "column": [
                            "*"
                        ],
                        "preSql": [
                            "truncate table warehouse"
                        ],
                        "connection": [
                            {
                                "jdbcUrl": "||_dsc_ob10_dsc_||obce-single:sys||_dsc_ob10_dsc_||jdbc:oceanbase://127.0.0.1:2883/tpcc",
                                "table": [
                                    "warehouse"
                                ]
                            }
                        ],
                        "batchSize": 1024,
                        "activeMemPercent": "90"
                    }
                }
            }
        ]
    }
}
```

**执行迁移**

```java
[root@obdocker datax]# python bin/datax.py job/mysql2ob.json

DataX (DATAX-OPENSOURCE-3.0), From Alibaba !
Copyright (C) 2010-2017, Alibaba Group. All Rights Reserved.


2022-10-05 14:09:03.456 [main] INFO  MessageSource - JVM TimeZone: GMT+08:00, Locale: zh_CN
2022-10-05 14:09:03.457 [main] INFO  MessageSource - use Locale: zh_CN timeZone: sun.util.calendar.ZoneInfo[id="GMT+08:00",offset=28800000,dstSavings=0,useDaylight=false,transitions=0,lastRule=null]
2022-10-05 14:09:03.475 [main] INFO  VMInfo - VMInfo# operatingSystem class => sun.management.OperatingSystemImpl
2022-10-05 14:09:03.479 [main] INFO  Engine - the machine info  =>

	osInfo:	Red Hat, Inc. 1.8 25.345-b01
	jvmInfo:	Linux amd64 3.10.0-1160.76.1.el7.x86_64
	cpu num:	8

	totalPhysicalMemory:	-0.00G
	freePhysicalMemory:	-0.00G
	maxFileDescriptorCount:	-1
	currentOpenFileDescriptorCount:	-1

	GC Names	[PS MarkSweep, PS Scavenge]

	MEMORY_NAME                    | allocation_size                | init_size
	PS Eden Space                  | 256.00MB                       | 256.00MB
	Code Cache                     | 240.00MB                       | 2.44MB
	Compressed Class Space         | 1,024.00MB                     | 0.00MB
	PS Survivor Space              | 42.50MB                        | 42.50MB
	PS Old Gen                     | 683.00MB                       | 683.00MB
	Metaspace                      | -0.00MB                        | 0.00MB



2022-10-05 14:09:03.505 [main] WARN  Engine - prioriy set to 0, because NumberFormatException, the value is: null
2022-10-05 14:09:03.506 [main] INFO  PerfTrace - PerfTrace traceId=job_-1, isEnable=false, priority=0
2022-10-05 14:09:03.506 [main] INFO  JobContainer - DataX jobContainer starts job.
2022-10-05 14:09:03.507 [main] INFO  JobContainer - Set jobId = 0
2022-10-05 14:09:03.716 [job-0] INFO  OriginalConfPretreatmentUtil - Available jdbcUrl:jdbc:mysql://127.0.0.1:3306/tpcc?useUnicode=true&characterEncoding=utf8&useSSL=false&yearIsDateType=false&zeroDateTimeBehavior=convertToNull&tinyInt1isBit=false&rewriteBatchedStatements=true.
2022-10-05 14:09:03.717 [job-0] WARN  OriginalConfPretreatmentUtil - 您的配置文件中的列配置存在一定的风险. 因为您未配置读取数据库表的列，当您的表字段个数、类型有变动时，可能影响任务正确性甚至会运行出错。请检查您的配置并作出修改.
2022-10-05 14:09:03.744 [job-0] INFO  DBUtil - this is ob1_0 jdbc url.
2022-10-05 14:09:03.744 [job-0] INFO  DBUtil - this is ob1_0 jdbc url. user=obce-single:sys:root :url=jdbc:oceanbase://127.0.0.1:2883/tpcc
2022-10-05 14:09:03.887 [job-0] INFO  DbUtils - value for query [SHOW VARIABLES LIKE 'ob_compatibility_mode'] is [MYSQL]
2022-10-05 14:09:03.892 [job-0] INFO  DBUtil - this is ob1_0 jdbc url.
2022-10-05 14:09:03.893 [job-0] INFO  DBUtil - this is ob1_0 jdbc url. user=obce-single:sys:root :url=jdbc:oceanbase://127.0.0.1:2883/tpcc?yearIsDateType=false&zeroDateTimeBehavior=convertToNull&tinyInt1isBit=false&rewriteBatchedStatements=true
2022-10-05 14:09:03.906 [job-0] INFO  OriginalConfPretreatmentUtil - table:[warehouse] all columns:[
w_id,w_name,w_street_1,w_street_2,w_city,w_state,w_zip,w_tax,w_ytd
].
2022-10-05 14:09:03.906 [job-0] WARN  OriginalConfPretreatmentUtil - 您的配置文件中的列配置信息存在风险. 因为您配置的写入数据库表的列为*，当您的表字段个数、类型有变动时，可能影响任务正确性甚至会运行出错。请检查您的配置并作出修改.
2022-10-05 14:09:03.907 [job-0] INFO  OriginalConfPretreatmentUtil - Write data [
INSERT INTO %s (w_id,w_name,w_street_1,w_street_2,w_city,w_state,w_zip,w_tax,w_ytd) VALUES(?,?,?,?,?,?,?,?,?)
], which jdbcUrl like:[||_dsc_ob10_dsc_||obce-single:sys||_dsc_ob10_dsc_||jdbc:oceanbase://127.0.0.1:2883/tpcc?yearIsDateType=false&zeroDateTimeBehavior=convertToNull&tinyInt1isBit=false&rewriteBatchedStatements=true]
2022-10-05 14:09:03.908 [job-0] INFO  JobContainer - jobContainer starts to do prepare ...
2022-10-05 14:09:03.908 [job-0] INFO  JobContainer - DataX Reader.Job [mysqlreader] do prepare work .
2022-10-05 14:09:03.908 [job-0] INFO  JobContainer - DataX Writer.Job [oceanbasev10writer] do prepare work .
2022-10-05 14:09:03.908 [job-0] INFO  DBUtil - this is ob1_0 jdbc url.
2022-10-05 14:09:03.908 [job-0] INFO  DBUtil - this is ob1_0 jdbc url. user=obce-single:sys:root :url=jdbc:oceanbase://127.0.0.1:2883/tpcc?yearIsDateType=false&zeroDateTimeBehavior=convertToNull&tinyInt1isBit=false&rewriteBatchedStatements=true
2022-10-05 14:09:03.914 [job-0] INFO  CommonRdbmsWriter$Job - Begin to execute preSqls:[truncate table warehouse]. context info:||_dsc_ob10_dsc_||obce-single:sys||_dsc_ob10_dsc_||jdbc:oceanbase://127.0.0.1:2883/tpcc?yearIsDateType=false&zeroDateTimeBehavior=convertToNull&tinyInt1isBit=false&rewriteBatchedStatements=true.
2022-10-05 14:09:04.046 [job-0] INFO  DBUtil - this is ob1_0 jdbc url.
2022-10-05 14:09:04.046 [job-0] INFO  DBUtil - this is ob1_0 jdbc url. user=obce-single:sys:root :url=jdbc:oceanbase://127.0.0.1:2883/tpcc?yearIsDateType=false&zeroDateTimeBehavior=convertToNull&tinyInt1isBit=false&rewriteBatchedStatements=true
2022-10-05 14:09:04.053 [job-0] INFO  DbUtils - value for query [show variables like 'version'] is [3.1.1]
2022-10-05 14:09:04.054 [job-0] INFO  JobContainer - jobContainer starts to do split ...
2022-10-05 14:09:04.054 [job-0] INFO  JobContainer - Job set Channel-Number to 4 channels.
2022-10-05 14:09:04.056 [job-0] INFO  JobContainer - DataX Reader.Job [mysqlreader] splits to [1] tasks.
2022-10-05 14:09:04.056 [job-0] INFO  JobContainer - DataX Writer.Job [oceanbasev10writer] splits to [1] tasks.
2022-10-05 14:09:04.068 [job-0] INFO  JobContainer - jobContainer starts to do schedule ...
2022-10-05 14:09:04.071 [job-0] INFO  JobContainer - Scheduler starts [1] taskGroups.
2022-10-05 14:09:04.073 [job-0] INFO  JobContainer - Running by standalone Mode.
2022-10-05 14:09:04.076 [taskGroup-0] INFO  TaskGroupContainer - taskGroupId=[0] start [1] channels for [1] tasks.
2022-10-05 14:09:04.079 [taskGroup-0] INFO  Channel - Channel set byte_speed_limit to 2000000.
2022-10-05 14:09:04.079 [taskGroup-0] INFO  Channel - Channel set record_speed_limit to -1, No tps activated.
2022-10-05 14:09:04.084 [taskGroup-0] INFO  TaskGroupContainer - taskGroup[0] taskId[0] attemptCount[1] is started
2022-10-05 14:09:04.086 [0-0-0-writer] INFO  OceanBaseV10Writer$Task - tableNumber:1,writerTask Class:com.alibaba.datax.plugin.writer.oceanbasev10writer.task.ConcurrentTableWriterTask
2022-10-05 14:09:04.087 [0-0-0-writer] INFO  ConcurrentTableWriterTask - configure url is unavailable, use obclient for connections.
2022-10-05 14:09:04.087 [0-0-0-reader] INFO  CommonRdbmsReader$Task - Begin to read record by Sql: [select * from warehouse
] jdbcUrl:[jdbc:mysql://127.0.0.1:3306/tpcc?useUnicode=true&characterEncoding=utf8&useSSL=false&yearIsDateType=false&zeroDateTimeBehavior=convertToNull&tinyInt1isBit=false&rewriteBatchedStatements=true].
2022-10-05 14:09:04.095 [0-0-0-writer] INFO  ConcurrentTableWriterTask - Disable partition calculation feature.
2022-10-05 14:09:04.097 [0-0-0-reader] INFO  CommonRdbmsReader$Task - Finished read record by Sql: [select * from warehouse
] jdbcUrl:[jdbc:mysql://127.0.0.1:3306/tpcc?useUnicode=true&characterEncoding=utf8&useSSL=false&yearIsDateType=false&zeroDateTimeBehavior=convertToNull&tinyInt1isBit=false&rewriteBatchedStatements=true].
2022-10-05 14:09:04.102 [0-0-0-writer] INFO  CommonRdbmsWriter$Task - write mode: insert
2022-10-05 14:09:04.102 [0-0-0-writer] INFO  ConcurrentTableWriterTask - writeRecordSql :INSERT INTO warehouse (w_id,w_name,w_street_1,w_street_2,w_city,w_state,w_zip,w_tax,w_ytd) VALUES(?,?,?,?,?,?,?,?,?)
2022-10-05 14:09:04.103 [0-0-0-writer] INFO  DBUtil - this is ob1_0 jdbc url.
2022-10-05 14:09:04.103 [0-0-0-writer] INFO  DBUtil - this is ob1_0 jdbc url. user=obce-single:sys:root :url=jdbc:oceanbase://127.0.0.1:2883/tpcc?yearIsDateType=false&zeroDateTimeBehavior=convertToNull&tinyInt1isBit=false&rewriteBatchedStatements=true
2022-10-05 14:09:04.108 [0-0-0-writer] ERROR ConcurrentTableWriterTask - partCalculator is null
2022-10-05 14:09:04.108 [0-0-0-writer] INFO  ConcurrentTableWriterTask - start 1 insert task.
2022-10-05 14:09:04.115 [0-0-0-writer] INFO  DBUtil - this is ob1_0 jdbc url.
2022-10-05 14:09:04.115 [0-0-0-writer] INFO  DBUtil - this is ob1_0 jdbc url. user=obce-single:sys:root :url=jdbc:oceanbase://127.0.0.1:2883/tpcc?yearIsDateType=false&zeroDateTimeBehavior=convertToNull&tinyInt1isBit=false&rewriteBatchedStatements=true
2022-10-05 14:09:04.122 [0-0-0-writer] INFO  ColumnMetaCache - fetch columnMeta of table warehouse success
2022-10-05 14:09:04.236 [0-0-0-writer] INFO  CommonRdbmsWriter$Task - isMemstoreFull=false
2022-10-05 14:09:04.237 [0-0-0-writer] INFO  ConcurrentTableWriterTask - ConcurrentTableWriter has put all task in queue, queueSize = 0,  total = 1, finished = 1
2022-10-05 14:09:04.285 [taskGroup-0] INFO  TaskGroupContainer - taskGroup[0] taskId[0] is successed, used[201]ms
2022-10-05 14:09:04.285 [taskGroup-0] INFO  TaskGroupContainer - taskGroup[0] completed it's tasks.
2022-10-05 14:09:14.084 [job-0] INFO  StandAloneJobContainerCommunicator - Total 2 records, 151 bytes | Speed 15B/s, 0 records/s | Error 0 records, 0 bytes |  All Task WaitWriterTime 0.000s |  All Task WaitReaderTime 0.000s | Percentage 100.00%
2022-10-05 14:09:14.084 [job-0] INFO  AbstractScheduler - Scheduler accomplished all tasks.
2022-10-05 14:09:14.085 [job-0] INFO  JobContainer - DataX Writer.Job [oceanbasev10writer] do post work.
2022-10-05 14:09:14.085 [job-0] INFO  JobContainer - DataX Reader.Job [mysqlreader] do post work.
2022-10-05 14:09:14.085 [job-0] INFO  JobContainer - DataX jobId [0] completed successfully.
2022-10-05 14:09:14.086 [job-0] INFO  HookInvoker - No hook invoked, because base dir not exists or is a file: /root/datax/hook
2022-10-05 14:09:14.087 [job-0] INFO  JobContainer -
	 [total cpu info] =>
		averageCpu                     | maxDeltaCpu                    | minDeltaCpu
		-1.00%                         | -1.00%                         | -1.00%


	 [total gc info] =>
		 NAME                 | totalGCCount       | maxDeltaGCCount    | minDeltaGCCount    | totalGCTime        | maxDeltaGCTime     | minDeltaGCTime
		 PS MarkSweep         | 0                  | 0                  | 0                  | 0.000s             | 0.000s             | 0.000s
		 PS Scavenge          | 0                  | 0                  | 0                  | 0.000s             | 0.000s             | 0.000s

2022-10-05 14:09:14.087 [job-0] INFO  JobContainer - PerfTrace not enable!
2022-10-05 14:09:14.087 [job-0] INFO  StandAloneJobContainerCommunicator - Total 2 records, 151 bytes | Speed 15B/s, 0 records/s | Error 0 records, 0 bytes |  All Task WaitWriterTime 0.000s |  All Task WaitReaderTime 0.000s | Percentage 100.00%
2022-10-05 14:09:14.088 [job-0] INFO  JobContainer -
任务启动时刻                    : 2022-10-05 14:09:03
任务结束时刻                    : 2022-10-05 14:09:14
任务总计耗时                    :                 10s
任务平均流量                    :               15B/s
记录写入速度                    :              0rec/s
读出记录总数                    :                   2
读写失败总数                    :                   0
```

**迁移完成，进行数据对比**

https://github.com/acquaai/obce/datax_oms.jpg


## DataX 的`channel bps`错误

DataX 安装后首次测试运行，可能遇到 **channel 的 bps 值为空**

```java
[root@obdocker datax]# python bin/datax.py job/job.json
......
经DataX智能分析,该任务最可能的错误原因是:
com.alibaba.datax.common.exception.DataXException: Code:[Framework-03], Description:[DataX引擎配置错误，该问题通常是由于DataX安装错误引起，请联系您的运维解决 .].  - 在有总bps限速条件下，单个channel的bps值不能为空，也不能为非正数
	at com.alibaba.datax.common.exception.DataXException.asDataXException(DataXException.java:30)
	at com.alibaba.datax.core.job.JobContainer.adjustChannelNumber(JobContainer.java:430)
	at com.alibaba.datax.core.job.JobContainer.split(JobContainer.java:387)
	at com.alibaba.datax.core.job.JobContainer.start(JobContainer.java:117)
	at com.alibaba.datax.core.Engine.start(Engine.java:93)
	at com.alibaba.datax.core.Engine.entry(Engine.java:175)
	at com.alibaba.datax.core.Engine.main(Engine.java:208)
```

**处理办法:**

修改 `core -> transport -> channel -> speed -> "byte": 2000000，将单个 channel 的大小改为 2MB 即可`。



**Ref:**

+ [OB使用指南-数据迁移](https://www.oceanbase.com/docs/community-observer-cn-10000000000449924)
+ [DataX](https://github.com/alibaba/DataX)
+ [OB B站视频](https://www.bilibili.com/video/BV1TY411h7XB/?spm_id_from=333.999.0.0&vd_source=ea963c63a0921a202a9b1b0e11112f1e)
