熟悉手动部署 OceanBase 集群能在 OBD 功能不满足需求时，可自写程序、脚本来部署满足需求的集群，也利于集群问题诊断。


## 部署规划

+ 系统资源

资源类型 | 阿里云 ECS
:-: | :-:
CPU | 4C
MEM | 16GB
Disk1 | /dev/vda, 100GB
Disk2 | /dev/vdb, 100GB

>建议单独磁盘挂载:
>+ /data, ext4
>+ /redo, ext4
>+ /home/admin, ext4

+ 角色规划

实验第一阶段部署 **1-1-1** 3 节点集群，第二阶段扩容集群为 **2-2-2** 6 节点集群，最后将集群缩容为 **1-1-1**。

IP | Roles | Notes
:-: | :-: | :-:
172.16.120.164 | OBD, OBProxy | Control, Deploy
172.16.120.165 | OBServer11 | OB Zone1
172.16.120.166 | OBServer21 | OB Zone2
172.16.120.167 | OBServer31 | OB Zone3
172.16.120.171 | OBServer12 | OB Zone1
172.16.120.172 | OBServer22 | OB Zone2
172.16.120.173 | OBServer32 | OB Zone3


## 部署 1-1-1 3节点集群

### 系统初始化（略）

>注意: 所有节点 NTP 的时间同步。

### 分发和部署软件

**OBD 节点安装**

```sh
[admin@OBD ~]$ sudo rpm -ivh libobclient-2.0.2-2.el7.x86_64.rpm
[admin@OBD ~]$ sudo rpm -ivh obclient-2.0.2-3.el7.x86_64.rpm
```

**OBServer 节点安装**

```sh
[admin@OBD ~]$ IPS='172.16.120.165 172.16.120.166 172.16.120.167'
[admin@OBD ~]$ for ip in $IPS; do echo $ip; scp oceanbase-ce-3.1.4-10000092022071511.el7.x86_64.rpm oceanbase-ce-libs-3.1.4-10000092022071511.el7.x86_64.rpm admin@$ip:~/; done

[admin@OBD ~]$ for ip in $IPS; do echo $ip; ssh admin@$ip "sudo rpm -Uvh oceanbase-ce*.rpm"; done
[admin@OBD ~]$ for ip in $IPS; do echo $ip; ssh admin@$ip "sudo chown -R admin:admin /home/admin/oceanbase /data /redo"; done
```

### 准备目录

```sh
for ip in $IPS; do echo $ip; ssh admin@$ip "mkdir -p /home/admin/oceanbase/store/obdemo /data/obdemo/sstable /redo/obdemo/{clog,ilog,slog}"; done

for ip in $IPS; do echo $ip; ssh admin@$ip "ln -s /data/obdemo/sstable /home/admin/oceanbase/store/obdemo/sstable"; done

for ip in $IPS; do echo $ip; ssh admin@$ip "ln -s /redo/obdemo/clog /home/admin/oceanbase/store/obdemo/clog"; done

for ip in $IPS; do echo $ip; ssh admin@$ip "ln -s /redo/obdemo/slog /home/admin/oceanbase/store/obdemo/slog"; done

for ip in $IPS; do echo $ip; ssh admin@$ip "ln -s /redo/obdemo/ilog /home/admin/oceanbase/store/obdemo/ilog"; done

for ip in $IPS; do echo $ip; ssh admin@$ip "echo 'export LD_LIBRARY_PATH=\$LD_LIBRARY_PATH:~/oceanbase/lib' >> ~/.bash_profile"; done
```

**OBServer 目录结构**

```sh
tree ~/oceanbase/store/ /data/ /redo/
/home/admin/oceanbase/store/
|-- obdemo
    |-- clog -> /redo/obdemo/clog
    |-- ilog -> /redo/obdemo/ilog
    |-- slog -> /redo/obdemo/slog
    |-- sstable -> /data/obdemo/sstable
/data/
|-- obdemo
    |-- sstable
/redo/
|-- obdemo
    |-- clog
    |-- ilog
    |-- slog
```


### 启动 OBServer 进程

```sh
ssh ob11

cd ~/oceanbase && bin/observer -i eth0 -p 2881 -P 2882 -z zone1 -d ~/oceanbase/store/obdemo -r '172.16.120.165:2882:2881;172.16.120.166:2882:2881;172.16.120.167:2882:2881' -c 20221004 -n obdemo -o "memory_limit=8G,cache_wash_threshold=1G,__min_full_resource_pool_memory=268435456,system_memory=3G,memory_chunk_cache_size=128M,cpu_count=16,net_thread_count=4,datafile_size=50G,stack_size=1536K"

ssh ob21

cd ~/oceanbase && bin/observer -i eth0 -p 2881 -P 2882 -z zone2 -d ~/oceanbase/store/obdemo -r '172.16.120.165:2882:2881;172.16.120.166:2882:2881;172.16.120.167:2882:2881' -c 20221004 -n obdemo -o "memory_limit=8G,cache_wash_threshold=1G,__min_full_resource_pool_memory=268435456,system_memory=3G,memory_chunk_cache_size=128M,cpu_count=16,net_thread_count=4,datafile_size=50G,stack_size=1536K"

ssh ob31

cd ~/oceanbase && bin/observer -i eth0 -p 2881 -P 2882 -z zone3 -d ~/oceanbase/store/obdemo -r '172.16.120.165:2882:2881;172.16.120.166:2882:2881;172.16.120.167:2882:2881' -c 20221004 -n obdemo -o "memory_limit=8G,cache_wash_threshold=1G,__min_full_resource_pool_memory=268435456,system_memory=3G,memory_chunk_cache_size=128M,cpu_count=16,net_thread_count=4,datafile_size=50G,stack_size=1536K"
```


### 初始化 OB 集群

```sql
# 初始密码为空
[admin@OBD ~]$ obclient -h172.16.120.165 -uroot -P2881 -p -c -A

obclient [(none)]> set session ob_query_timeout=1000000000; alter system bootstrap ZONE 'zone1' SERVER '172.16.120.165:2882', ZONE 'zone2' SERVER '172.16.120.166:2882', ZONE 'zone3' SERVER '172.16.120.167:2882';
```

修改 sys 租户 root 用户密码，初始密码为空。

```sql
[admin@OBD ~]$ obclient -h172.16.120.165 -uroot@sys -P2881 -p -c -A oceanbase

obclient [oceanbase]> alter user root identified by 'rootPWD123';
```

>常见 bootstrap 失败原因
>+ 集群里 OBServer 节点之间网络延时超过 200ms（建议在 30ms 以内）
>+ 集群里 OBServer 节点之间时间同步误差超过 100ms（建议在 5ms 以内）
>+ 主机 ulimit 会话限制没有修改或生效
>+ 主机内核参数 sysctl.conf 没有修改或生效
>+ OBServer 启动用户不对，建议 admin
>+ OBServer 启动目录不对，必须是 ~/oceanbase
>+ OBServer 的可用内存低于 8G
>+ OBServer 的事务日志目录可用空间低于 5%
>+ OBServer 启动参数不对（zone 名称对不上，rootservice_list 地址或格式不对，集群名不统一等）


### 部署 OBProxy

**安装 OBProxy 包**

```sh
[admin@OBD obdemo]$ sudo rpm -ivh obproxy-ce-3.2.3.5-2.el7.x86_64.rpm
```

**启动 OBProxy 进程**

```sh
[admin@OBD ~]$ cd ~/obproxy-3.2.3.5 && bin/obproxy -r "172.16.120.165:2881;172.16.120.166:2881;172.16.120.167:2881" -p 2883 -o "enable_strict_kernel_release=false,enable_cluster_checkout=false,enable_metadb_used=false" -c obdemo
```

**查看 OBProxy 监听**

```sh
[admin@OBD obproxy-3.2.3.5]$ netstat -ntlp |grep obproxy
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
tcp        0      0 0.0.0.0:2883            0.0.0.0:*               LISTEN      13596/bin/obproxy
tcp        0      0 0.0.0.0:2884            0.0.0.0:*               LISTEN      13596/bin/obproxy
```

**修改 obproxy 管理员密码，初始密码为空**

```sql
obclient -h172.16.120.164 -uroot@proxysys -P2883 -p

alter proxyconfig set obproxy_sys_password='proxySYSPWD123';
```

**修改 obproxy 与 observer 通信用户密码**

```sql
alter proxyconfig set observer_sys_password='proxyROPWD123';
```

**创建 obproxy 和 observer 通讯用户**

```sql
obclient -h172.16.120.165 -uroot@sys -P2881 -prootPWD123 -c -A oceanbase
grant select on oceanbase.* to proxyro identified by 'proxyROPWD123';

obclient [oceanbase]> show full processlist;
+------------+---------+--------+----------------------+-----------+---------+------+--------+-----------------------+----------------+------+----------------------+
| Id         | User    | Tenant | Host                 | db        | Command | Time | State  | Info                  | Ip             | Port | Proxy_sessid         |
+------------+---------+--------+----------------------+-----------+---------+------+--------+-----------------------+----------------+------+----------------------+
| 3221498778 | proxyro | sys    | 172.16.120.164:13980 | oceanbase | Sleep   |    7 | SLEEP  | NULL                  | 172.16.120.165 | 2881 | 12398542420109885447 |
| 3221498765 | root    | sys    | 172.16.120.164:13970 | oceanbase | Query   |    0 | ACTIVE | show full processlist | 172.16.120.165 | 2881 | 12398542420109885446 |
+------------+---------+--------+----------------------+-----------+---------+------+--------+-----------------------+----------------+------+----------------------+
2 rows in set (0.022 sec)

obclient [oceanbase]> show proxyconfig like '%_sys_password%';
```

>常见 OBProxy 连接失败原因:
>+ 启动用户不对，建议 admin
>+ 启动目录不对，必须是 `~/obproxy-<version>`
>+ proxyro 的用户需要在 OB 集群里创建，并建议设置密码
>+ proxyro 的密码必须在 OBProxy 进程里通过参数 `observer_sys_password` 设置
>+ OBProxy 启动时指定的集群名（-c）跟 OB 集群里的 Cluster 不一致


### 创建业务租户、业务数据库和表

**创建业务租户**

```sql
obclient -h172.16.120.164 -uroot@sys#obdemo -P2883 -prootPWD123 -c -A oceanbase

CREATE RESOURCE UNIT 4c2g
       max_cpu 4,
       min_cpu 2,
       max_memory '2G',
       min_memory '2G',
       max_iops 1000,
       min_iops 128,
       max_disk_size '10G',
       max_session_num 300
     ;

CREATE RESOURCE POOL rp1
        UNIT '4c2g',
        UNIT_NUM = 1,
        ZONE_LIST = ('zone1','zone2','zone3')
      ;

CREATE TENANT IF NOT EXISTS my_1st_tenant
      CHARSET='utf8mb4',
      ZONE_LIST= ('zone1','zone2','zone3'), 
      PRIMARY_ZONE='zone1', 
      RESOURCE_POOL_LIST=('rp1') 
    SET ob_tcp_invited_nodes='%' 
    ;
```

**创建业务数据库**

```sql
obclient -h172.16.120.164  -uroot@my_1st_tenant -P2883 -c -A oceanbase

obclient [oceanbase]> create databases  my_1st_db;
+--------------------+
| Database           |
+--------------------+
| oceanbase          |
| information_schema |
| mysql              |
| test               |
| my_1st_db          |
+--------------------+
5 rows in set (0.012 sec)

obclient [oceanbase]> grant all privileges on my_1st_db.* to my_1st_user identified by 'password';
```

**创建业务表、插入数据**

```sql
obclient -h172.16.120.164 -umy_1st_user@my_1st_tenant#obdemo -P2883 -ppassword -c -A my_1st_db;  

obclient [my_1st_db]>  create table my_1st_table (id int, name varchar(10));

obclient [my_1st_db]> desc my_1st_table;
+-------+-------------+------+-----+---------+-------+
| Field | Type        | Null | Key | Default | Extra |
+-------+-------------+------+-----+---------+-------+
| id    | int(11)     | YES  |     | NULL    |       |
| name  | varchar(10) | YES  |     | NULL    |       |
+-------+-------------+------+-----+---------+-------+
2 rows in set (0.018 sec)

obclient [my_1st_db]> insert into my_1st_table values (1, 'hi,ob');
Query OK, 1 row affected (0.008 sec)

obclient [my_1st_db]> select * from my_1st_db.my_1st_table;
+------+-------+
| id   | name  |
+------+-------+
|    1 | hi,ob |
+------+-------+
1 row in set (0.001 sec)
```


##  扩容集群为 2-2-2 6 节点

**1-1-1 集群节点**

```sql
obclient [oceanbase]> SELECT * FROM __all_server;
+----------------------------+----------------------------+----------------+----------+----+-------+------------+-----------------+--------+-----------------------+----------------------------------------------------------------------------------------+-----------+--------------------+--------------+----------------+-------------------+
| gmt_create                 | gmt_modified               | svr_ip         | svr_port | id | zone  | inner_port | with_rootserver | status | block_migrate_in_time | build_version                                                                          | stop_time | start_service_time | first_sessid | with_partition | last_offline_time |
+----------------------------+----------------------------+----------------+----------+----+-------+------------+-----------------+--------+-----------------------+----------------------------------------------------------------------------------------+-----------+--------------------+--------------+----------------+-------------------+
| 2022-10-04 09:13:30.059362 | 2022-10-04 09:13:41.647889 | 172.16.120.165 |     2882 |  1 | zone1 |       2881 |               1 | active |                     0 | 3.1.4_10000092022071511-b4bfa011ceaef428782dcb65ae89190c40b78c2f(Jul 15 2022 11:45:14) |         0 |   1664846018647328 |            0 |              1 |                 0 |
| 2022-10-04 09:13:28.904073 | 2022-10-04 09:13:43.656348 | 172.16.120.166 |     2882 |  2 | zone2 |       2881 |               0 | active |                     0 | 3.1.4_10000092022071511-b4bfa011ceaef428782dcb65ae89190c40b78c2f(Jul 15 2022 11:45:14) |         0 |   1664846019657473 |            0 |              1 |                 0 |
| 2022-10-04 09:13:28.898884 | 2022-10-04 09:13:40.386624 | 172.16.120.167 |     2882 |  3 | zone3 |       2881 |               0 | active |                     0 | 3.1.4_10000092022071511-b4bfa011ceaef428782dcb65ae89190c40b78c2f(Jul 15 2022 11:45:14) |         0 |   1664846020375659 |            0 |              1 |                 0 |
+----------------------------+----------------------------+----------------+----------+----+-------+------------+-----------------+--------+-----------------------+----------------------------------------------------------------------------------------+-----------+--------------------+--------------+----------------+-------------------+
3 rows in set (0.011 sec)
```

### 新节点初始化

### 新节点启动 OBServer 进程

```sh
ssh ob12
cd ~/oceanbase && bin/observer -i eth0 -p 2881 -P 2882 -z zone1 -d ~/oceanbase/store/obdemo -r '172.16.120.171:2882:2881;172.16.120.172:2882:2881;172.16.120.173:2882:2881' -c 20221004 -n obdemo -o "memory_limit=8G,cache_wash_threshold=1G,__min_full_resource_pool_memory=268435456,system_memory=3G,memory_chunk_cache_size=128M,cpu_count=16,net_thread_count=4,datafile_size=50G,stack_size=1536K"

ssh ob22
cd ~/oceanbase && bin/observer -i eth0 -p 2881 -P 2882 -z zone2 -d ~/oceanbase/store/obdemo -r '172.16.120.171:2882:2881;172.16.120.172:2882:2881;172.16.120.173:2882:2881' -c 20221004 -n obdemo -o "memory_limit=8G,cache_wash_threshold=1G,__min_full_resource_pool_memory=268435456,system_memory=3G,memory_chunk_cache_size=128M,cpu_count=16,net_thread_count=4,datafile_size=50G,stack_size=1536K"

ssh ob32
cd ~/oceanbase && bin/observer -i eth0 -p 2881 -P 2882 -z zone3 -d ~/oceanbase/store/obdemo -r '172.16.120.171:2882:2881;172.16.120.172:2882:2881;172.16.120.173:2882:2881' -c 20221004 -n obdemo -o "memory_limit=8G,cache_wash_threshold=1G,__min_full_resource_pool_memory=268435456,system_memory=3G,memory_chunk_cache_size=128M,cpu_count=16,net_thread_count=4,datafile_size=50G,stack_size=1536K"
```

### 向 ZONE 内添加 OBServer 节点

```sql
obclient [oceanbase]> alter system add  server '172.16.120.171:2882' zone 'zone1';
Query OK, 0 rows affected (0.007 sec)

obclient [oceanbase]> alter system add  server '172.16.120.172:2882' zone 'zone2';
Query OK, 0 rows affected (0.017 sec)

obclient [oceanbase]> alter system add  server '172.16.120.173:2882' zone 'zone3';
Query OK, 0 rows affected (0.006 sec)
```

>注意: IP:Port，Port 为 RPC 端口。

**查看集群现有资源**

```sql
obclient [oceanbase]> select zone,concat(svr_ip,':',svr_port) observer, cpu_capacity,cpu_total,cpu_assigned,cpu_assigned_percent, mem_capacity,mem_total,mem_assigned,mem_assigned_percent, unit_Num,round(`load`,2) `load`, round(cpu_weight,2) cpu_weight, round(memory_weight,2) mem_weight, leader_count from __all_virtual_server_stat order by zone,svr_ip;
+-------+---------------------+--------------+-----------+--------------+----------------------+--------------+------------+--------------+----------------------+----------+------+------------+------------+--------------+
| zone  | observer            | cpu_capacity | cpu_total | cpu_assigned | cpu_assigned_percent | mem_capacity | mem_total  | mem_assigned | mem_assigned_percent | unit_Num | load | cpu_weight | mem_weight | leader_count |
+-------+---------------------+--------------+-----------+--------------+----------------------+--------------+------------+--------------+----------------------+----------+------+------------+------------+--------------+
| zone1 | 172.16.120.165:2882 |           16 |        16 |          4.5 |                   28 |   5368709120 | 5368709120 |   3489660928 |                   65 |        2 | 0.54 |       0.30 |       0.70 |         1323 |
| zone1 | 172.16.120.171:2882 |           16 |        16 |            0 |                    0 |   5368709120 | 5368709120 |            0 |                    0 |        0 | 0.00 |       0.30 |       0.70 |            0 |
| zone2 | 172.16.120.166:2882 |           16 |        16 |          4.5 |                   28 |   5368709120 | 5368709120 |   3489660928 |                   65 |        2 | 0.54 |       0.30 |       0.70 |            0 |
| zone2 | 172.16.120.172:2882 |           16 |        16 |            0 |                    0 |   5368709120 | 5368709120 |            0 |                    0 |        0 | 0.00 |       0.30 |       0.70 |            0 |
| zone3 | 172.16.120.167:2882 |           16 |        16 |          4.5 |                   28 |   5368709120 | 5368709120 |   3489660928 |                   65 |        2 | 0.54 |       0.30 |       0.70 |            0 |
| zone3 | 172.16.120.173:2882 |           16 |        16 |            0 |                    0 |   5368709120 | 5368709120 |            0 |                    0 |        0 | 0.00 |       0.30 |       0.70 |            0 |
+-------+---------------------+--------------+-----------+--------------+----------------------+--------------+------------+--------------+----------------------+----------+------+------------+------------+--------------+
6 rows in set (0.013 sec)
```

**调大 UNIT_NUM**

```sql
obclient [oceanbase]> alter resource pool rp1 unit_num=2;
Query OK, 0 rows affected (0.029 sec)
```

**2-2-2 集群节点**

```sql
obclient [oceanbase]> SELECT * FROM __all_server;
```


##  扩容集群为 2-2-2 6 节点

**调小 UNIT_NUM**

```sql
obclient [oceanbase]>  alter resource pool rp1 unit_num=1;
```

**集群资源变动会导致分区需要重新平衡，查看 job_status 列状态**

```sql
obclient [oceanbase]> select * from __all_rootservice_job;
+----------------------------+----------------------------+--------+-------------------------------+------------+-------------+----------+-----------+-------------+-------------+---------------+----------+------------+--------------+--------+----------+---------+----------------+-------------+----------+-----------------+------------------+---------------+-----------------+
| gmt_create                 | gmt_modified               | job_id | job_type                      | job_status | return_code | progress | tenant_id | tenant_name | database_id | database_name | table_id | table_name | partition_id | svr_ip | svr_port | unit_id | rs_svr_ip      | rs_svr_port | sql_text | extra_info      | resource_pool_id | tablegroup_id | tablegroup_name |
+----------------------------+----------------------------+--------+-------------------------------+------------+-------------+----------+-----------+-------------+-------------+---------------+----------+------------+--------------+--------+----------+---------+----------------+-------------+----------+-----------------+------------------+---------------+-----------------+
| 2022-10-04 12:27:05.565070 | 2022-10-04 12:27:05.565070 |      1 | SHRINK_RESOURCE_POOL_UNIT_NUM | INPROGRESS |        NULL |        0 |      1001 | NULL        |        NULL | NULL          |     NULL | NULL       |         NULL | NULL   |     NULL |    NULL | 172.16.120.165 |        2882 | NULL     | new_unit_num: 1 |             1002 |          NULL | NULL            |
+----------------------------+----------------------------+--------+-------------------------------+------------+-------------+----------+-----------+-------------+-------------+---------------+----------+------------+--------------+--------+----------+---------+----------------+-------------+----------+-----------------+------------------+---------------+-----------------+
1 row in set (0.001 sec)

obclient [oceanbase]> select * from __all_rootservice_job;
+----------------------------+----------------------------+--------+-------------------------------+------------+-------------+----------+-----------+-------------+-------------+---------------+----------+------------+--------------+--------+----------+---------+----------------+-------------+----------+-----------------+------------------+---------------+-----------------+
| gmt_create                 | gmt_modified               | job_id | job_type                      | job_status | return_code | progress | tenant_id | tenant_name | database_id | database_name | table_id | table_name | partition_id | svr_ip | svr_port | unit_id | rs_svr_ip      | rs_svr_port | sql_text | extra_info      | resource_pool_id | tablegroup_id | tablegroup_name |
+----------------------------+----------------------------+--------+-------------------------------+------------+-------------+----------+-----------+-------------+-------------+---------------+----------+------------+--------------+--------+----------+---------+----------------+-------------+----------+-----------------+------------------+---------------+-----------------+
| 2022-10-04 12:27:05.565070 | 2022-10-04 12:32:48.415491 |      1 | SHRINK_RESOURCE_POOL_UNIT_NUM | SUCCESS    |           0 |      100 |      1001 | NULL        |        NULL | NULL          |     NULL | NULL       |         NULL | NULL   |     NULL |    NULL | 172.16.120.165 |        2882 | NULL     | new_unit_num: 1 |             1002 |          NULL | NULL            |
+----------------------------+----------------------------+--------+-------------------------------+------------+-------------+----------+-----------+-------------+-------------+---------------+----------+------------+--------------+--------+----------+---------+----------------+-------------+----------+-----------------+------------------+---------------+-----------------+
1 row in set (0.001 sec)
```

**从 ZONE 中减少 OBServer 节点**

```sql
obclient [oceanbase]> alter system delete server '172.16.120.171:2882' zone 'zone1';
```

**等待 job 完成**

```sql
obclient [oceanbase]> select job_status from __all_rootservice_job;
+------------+
| job_status |
+------------+
| SUCCESS    |
| INPROGRESS |
+------------+
2 rows in set (0.000 sec)
```

>注意: **依次**减少剩余 OBServer 节点。

```sql
obclient [oceanbase]> alter system delete server '172.16.120.172:2882' zone 'zone2';
obclient [oceanbase]> alter system delete server '172.16.120.173:2882' zone 'zone3';
```


**Ref:**

+ [OceanBase 扩容和缩容概述](https://www.oceanbase.com/docs/community-observer-cn-10000000000450206)
