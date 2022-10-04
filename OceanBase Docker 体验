# OB Docker 体验

## 练习目的

本练习目的是通过 OceanBase Docker 容器，快速的体验 OceanBase 的自动化部署过程，及了解 OceanBase 集群安装成功后的目录特点和使用方法。

## 练习条件

+ 笔记本或服务器，OceanBase 可用内存不少于 12GB。
+ 操作系统不限，能安装 Docker 环境即可。

## 练习内容

1. 下载 Docker 镜像: [OceanBase 官方社区版镜像](https://hub.docker.com/r/oceanbase/oceanbase-ce)
2. 使用 OBD 命令完成 OceanBase 集群部署
3. 创建一个业务租户、一个业务数据库及一些表

## 练习环境

**操作系统**

```sh
[root@obdocker ~]# cat /etc/redhat-release 
CentOS Linux release 7.9.2009 (Core)
```

**CPU 资源: 8C**

```sh
[root@obdocker ~]# grep processor /proc/cpuinfo | wc -l
8
```

**Memory 资源: 16GB**

```sh
[root@obdocker ~]# lsmem | grep 'Total online memory'
Total online memory:      16G
```

**Memory 资源: 100GB，练习实际并不会使用这么多存储空间**

## 系统初始化配置

+ 个性主机名

```sh
[root@obdocker ~]# hostnamectl set-hostname obDocker --static --pretty --transient
```

+ 关闭 Selinux 和 Firewall

```sh
[root@obdocker ~]# sed -i s/^SELINUX=.*$/SELINUX=disabled/ /etc/selinux/config
[root@obdocker ~]# setenforce 0
[root@obdocker ~]# systemctl stop firewalld
[root@obdocker ~]# systemctl disable firewalld

+ 添加 Docker 仓库 repo

```sh
[root@obdocker ~]# yum install yum-utils -y
[root@obdocker ~]# yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
Loaded plugins: fastestmirror
adding repo from: https://download.docker.com/linux/centos/docker-ce.repo
grabbing file https://download.docker.com/linux/centos/docker-ce.repo to /etc/yum.repos.d/docker-ce.repo
repo saved to /etc/yum.repos.d/docker-ce.repo
```

+ 安装 Docker 并启动服务

```sh
[root@obdocker ~]# yum install docker-ce docker-ce-cli containerd.io -y
[root@obdocker ~]# systemctl start docker
[root@obdocker ~]# systemctl enable docker
Created symlink from /etc/systemd/system/multi-user.target.wants/docker.service to /usr/lib/systemd/system/docker.service.
```

+ 查看 Docker 版本

```sh
[root@obdocker ~]# docker version
Client: Docker Engine - Community
 Version:           20.10.18
 API version:       1.41
 Go version:        go1.18.6
 Git commit:        b40c2f6
 Built:             Thu Sep  8 23:14:08 2022
 OS/Arch:           linux/amd64
 Context:           default
 Experimental:      true

Server: Docker Engine - Community
 Engine:
  Version:          20.10.18
  API version:      1.41 (minimum version 1.12)
  Go version:       go1.18.6
  Git commit:       e42327a
  Built:            Thu Sep  8 23:12:21 2022
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          1.6.8
  GitCommit:        9cd3357b7fd7218e4aec3eae239db1f68a5a6ec6
 runc:
  Version:          1.1.4
  GitCommit:        v1.1.4-0-g5fd4c4d
 docker-init:
  Version:          0.19.0
  GitCommit:        de40ad0
```

## 拉取 OceanBase Docker 镜像

+ 搜索 OceanBase 镜像并拉取

```sh
[root@obdocker ~]# docker search oceanbase
NAME                             DESCRIPTION                                     STARS     OFFICIAL   AUTOMATED
oceanbase/oceanbase-ce           OceanBase is an open-source, distributed HTA…   10                   
obpilot/oceanbase-ce             3 steps to run an OceanBase-CE docker in you…   4                    
oceanbase/oceanbase-xe           OceanBase Database 2.2 Express Edition          4                    
oceanbase/obce-mini              obce-mini is a mini standalone test image fo…   3                    
oceanbase/obce-operator          obce-operator                                   1                    
zibuyu886/oceanbase-ce-cluster   OceanBase ce cluster                            1                    
oceanbase/miniob                 miniob database competition                     1                    
huweijie/oceanbase-ce-deploy                                                     0                    
oceanbase/obproxy-ce                                                             0                    
oceanbase/centos7                                                                0                    
oceanbasedev/oceanbase-cn                                                        0                    
oceanbasedev/obagent                                                             0                    
oceanbasedev/ob-operator                                                         0                    
superbigfu/oceanbase                                                             0                    
hongxinsun/oceanbase-cn                                                          0                    
hnwyllmm/oceanbase-ce-x86                                                        0                    
oceanbasedev/create_tenant                                                       0                    
oceanbasedev/obproxy                                                             0                    
hnwyllmm/oceanbase-ce                                                            0                    
hnwyllmm/oceanbase-ce-aarch64                                                    0                    
20220121/oceanbase                                                               0                    
jimmyzhou623/oceanbase                                                           0                    
oceanbasedev/sealer-ob                                                           0                    
hongxinsun/ob-operator           oceanbase operator                              0                    
hongweiqin/anolisos-oceanbase    A tentative deploy of oceanbase.                0                    
```

根据镜像描述，这里使用 **obpilot/oceanbase-ce** 镜像。

```sh
[root@obdocker ~]# docker pull obpilot/oceanbase-ce 
Using default tag: latest
latest: Pulling from obpilot/oceanbase-ce
7a0437f04f83: Pull complete 
615dc48ac9f1: Pull complete 
b10c1cdae3af: Pull complete 
4f4fb700ef54: Pull complete 
c0f6c94a6a6a: Pull complete 
792630f35e24: Pull complete 
Digest: sha256:7ac28415cf27ba19cb47acb67a55ebf9848ad73a63d80b7e2e85d653233dbaeb
Status: Downloaded newer image for obpilot/oceanbase-ce:latest
docker.io/obpilot/oceanbase-ce:latest

[root@obdocker ~]# docker images
REPOSITORY             TAG       IMAGE ID       CREATED         SIZE
obpilot/oceanbase-ce   latest    943379e0b05b   10 months ago   2.25GB
```

+ 启动 OceanBase 容器，指定该容器可用内存为 12GB

```sh
[root@obdocker ~]# docker run -itd -m 12G --name my-1st-ob obpilot/oceanbase-ce:latest
1d0b9893c55b06ea199f89ff8b5c9c94c3b854b6dfe9593e172cc6d5ecb57be9
```

+ 查看运行中的容器

```sh
[root@obdocker ~]# docker ps
CONTAINER ID   IMAGE                         COMMAND       CREATED          STATUS         PORTS                NAMES
1d0b9893c55b   obpilot/oceanbase-ce:latest   "/bin/bash"   10 seconds ago   Up 8 seconds   2881/tcp, 2883/tcp   my-1st-ob
```

## 启动 OceanBase 集群

**进入 OB 容器**

```sh
[root@obdocker ~]# docker exec -it my-1st-ob bash
```

**查看 OB 集群，obdemo 集群已部署，但未启动**

```sh
[admin@1d0b9893c55b ~]$ obd cluster list
+------------------------------------------------------------+
|                        Cluster List                        |
+--------+---------------------------------+-----------------+
| Name   | Configuration Path              | Status (Cached) |
+--------+---------------------------------+-----------------+
| obdemo | /home/admin/.obd/cluster/obdemo | deployed        |
+--------+---------------------------------+-----------------+
```

**启动 obdemo 集群**

```sh
[admin@1d0b9893c55b ~]$ obd cluster start obdemo
Get local repositories and plugins ok
Open ssh connection ok
Cluster param config check ok
Check before start observer ok
[WARN] (127.0.0.1) The recommended value of fs.aio-max-nr is 1048576 (Current value: 65536)

Check before start obproxy ok
Start observer ok
observer program health check ok
Connect to observer ok
Initialize cluster
Cluster bootstrap ok
Wait for observer init ok
+---------------------------------------------+
|                   observer                  |
+-----------+---------+------+-------+--------+
| ip        | version | port | zone  | status |
+-----------+---------+------+-------+--------+
| 127.0.0.1 | 3.1.1   | 2881 | zone1 | active |
+-----------+---------+------+-------+--------+

Start obproxy ok
obproxy program health check ok
Connect to obproxy ok
Initialize cluster
+---------------------------------------------+
|                   obproxy                   |
+-----------+------+-----------------+--------+
| ip        | port | prometheus_port | status |
+-----------+------+-----------------+--------+
| 127.0.0.1 | 2883 | 2884            | active |
+-----------+------+-----------------+--------+
obdemo running
```

至此，obdemo 集群已正常启动运行。

**查看 obdemo 集群相关配置信息**

```sh
[admin@1d0b9893c55b ~]$ obd cluster display obdemo
Get local repositories and plugins ok
Open ssh connection ok
Cluster status check ok
Connect to observer ok
Wait for observer init ok
+---------------------------------------------+
|                   observer                  |
+-----------+---------+------+-------+--------+
| ip        | version | port | zone  | status |
+-----------+---------+------+-------+--------+
| 127.0.0.1 | 3.1.1   | 2881 | zone1 | active |
+-----------+---------+------+-------+--------+

Connect to obproxy ok
+---------------------------------------------+
|                   obproxy                   |
+-----------+------+-----------------+--------+
| ip        | port | prometheus_port | status |
+-----------+------+-----------------+--------+
| 127.0.0.1 | 2883 | 2884            | active |
+-----------+------+-----------------+--------+
```

## 创建业务租户、数据库和表

创建业务租户、数据库及表前，需要使用 OceanBase-CE 版本（只支持 MySQL 模拟）的 MySQL 租户默认管理员用户名 **root** 登录 OceanBase。

由于该镜像已配置了 root 用户密码，先获取密码。

```sh
[admin@1d0b9893c55b ~]$  grep password /home/admin/.obd/cluster/obdemo/config.yaml 
#   password: your password if need
    root_password: rootPWD123 # root user password, can be empty
    proxyro_password:  proxyROPWD123 # proxyro user pasword, consistent with obproxy's observer_sys_password, can be empty
    obproxy_sys_password:  proxySYSPWD123 # obproxy sys user password, can be empty
    observer_sys_password: proxyROPWD123 # proxyro user pasword, consistent with oceanbase-ce's proxyro_password, can be empty
```

**获取 OceanBase 集群名称:**

```sh
[admin@1d0b9893c55b ~]$ grep appname /home/admin/.obd/cluster/obdemo/config.yaml         
    appname: obce-single
    # observer cluster name, consistent with oceanbase-ce's appname
```

对 MySQL 模式，使用 MySQL 客户端或 obclient 客户端都可以连接 MySQL 租户。租户的连接帐户格式: **用户名@租户名#集群名**； **集群名:租户名:用户名**。MySQL 租户的管理员用户名默认是 root。

```sh
[admin@1d0b9893c55b ~]$ obclient -h127.0.0.1 -uroot@sys#obce-single -P2883 -prootPWD123 -c -A oceanbase             
Welcome to the OceanBase.  Commands end with ; or \g.
Your MySQL connection id is 10
Server version: 5.6.25 OceanBase 3.1.1 (r4-8c615943cbd25a6f7b8bdfd8677a13a21709a05e) (Built Oct 21 2021 10:52:05)

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MySQL [oceanbase]> 
```

**获取 OceanBase 可用资源:**

```sql
MySQL [oceanbase]> select svr_ip,svr_port,cpu_total,mem_total,disk_total,zone from __all_virtual_server_stat;
+-----------+----------+-----------+------------+------------+-------+
| svr_ip    | svr_port | cpu_total | mem_total  | disk_total | zone  |
+-----------+----------+-----------+------------+------------+-------+
| 127.0.0.1 |     2882 |        14 | 4294967296 | 5368709120 | zone1 |
+-----------+----------+-----------+------------+------------+-------+
1 row in set (0.004 sec)


MySQL [oceanbase]> select sum(uc.max_cpu), sum(uc.max_memory) from __all_resource_pool as rp, __all_unit_config as uc where rp.unit_config_id=uc.unit_config_id;             
+-----------------+--------------------+
| sum(uc.max_cpu) | sum(uc.max_memory) |
+-----------------+--------------------+
|               9 |         3435973836 |
+-----------------+--------------------+
1 row in set (0.001 sec)
```


**创建一个 4 CPU，2GB 内存的资源规格:**

```sql
MySQL [oceanbase]> CREATE RESOURCE UNIT 4c2g
       max_cpu 4,
       min_cpu 2,
       max_memory '2G',
       min_memory '2G',
       max_iops 1000,
       min_iops 128,
       max_disk_size '10G',
       max_session_num 300
     ;
```

**使用 4c2g 资源规格创建资源池:**

```sql
MySQL [oceanbase]> CREATE RESOURCE POOL rp1
        UNIT '4c2g',
        UNIT_NUM = 1,
        ZONE_LIST = ('zone1')
      ;
```

**使用 rp1 资源池创建业务租户:**

```sql
MySQL [oceanbase]> CREATE TENANT IF NOT EXISTS my_1st_tenant
      CHARSET='utf8mb4',
      ZONE_LIST=('zone1'), 
      PRIMARY_ZONE='RANDOM', 
      RESOURCE_POOL_LIST=('rp1') 
    SET ob_tcp_invited_nodes='%' 
    ;
```

**创建业务租户数据库:**

```sql
[admin@1d0b9893c55b ~]$ obclient -h127.0.0.1 -uroot@my_1st_tenant -P2881 -c -A oceanbase


MySQL [oceanbase]> create database my_1st_db;
Query OK, 1 row affected (0.011 sec)


MySQL [oceanbase]> show databases;
+--------------------+
| Database           |
+--------------------+
| oceanbase          |
| information_schema |
| mysql              |
| test               |
| my_1st_db          |
+--------------------+
5 rows in set (0.002 sec)
```


**创建业务租户用户，并授权访问数据库所有对象:**

```sql
MySQL [oceanbase]> grant all privileges on my_1st_db.* to my_1st_user identified by 'password';
Query OK, 0 rows affected (0.020 sec)


MySQL [oceanbase]> show grants for my_1st_user;
+--------------------------------------------------------+
| Grants for my_1st_user@%                               |
+--------------------------------------------------------+
| GRANT USAGE ON *.* TO 'my_1st_user'                    |
| GRANT ALL PRIVILEGES ON `my_1st_db`.* TO 'my_1st_user' |
+--------------------------------------------------------+
2 rows in set (0.008 sec)
```


**使用业务用户登录，创建业务表:**

```sql
[admin@1d0b9893c55b ~]$  obclient -h127.0.0.1 -umy_1st_user@my_1st_tenant -P2881 -ppassword -c -A my_1st_db;
或
[admin@1d0b9893c55b ~]$ obclient -h127.0.0.1 -umy_1st_user@my_1st_tenant#obce-single -P2883 -ppassword -c -A my_1st_db;  


MySQL [my_1st_db]> create table my_1st_table (id int, name varchar(10));
Query OK, 0 rows affected (0.039 sec)

MySQL [my_1st_db]> desc my_1st_table;
+-------+-------------+------+-----+---------+-------+
| Field | Type        | Null | Key | Default | Extra |
+-------+-------------+------+-----+---------+-------+
| id    | int(11)     | YES  |     | NULL    |       |
| name  | varchar(10) | YES  |     | NULL    |       |
+-------+-------------+------+-----+---------+-------+
2 rows in set (0.002 sec)
```


**插入业务数据:**

```sql
MySQL [my_1st_db]> insert into my_1st_table values (1, 'hi,ob');
Query OK, 1 row affected (0.008 sec)

MySQL [my_1st_db]> select * from my_1st_db.my_1st_table;
+------+-------+
| id   | name  |
+------+-------+
|    1 | hi,ob |
+------+-------+
1 row in set (0.001 sec)
```


Ref:

+ [Linux 安装 Docker](https://docs.docker.com/engine/install/centos/)
+ 社区版官网-文档-学习中心-入门教程：[实战教程第二章2.2：如何快速体验 OceanBase](https://open.oceanbase.com/docs/tutorials/quickstart/V1.0.0/2-2-how-to-quickly-experience-oceanbase)
+ 社区版官网-博客-入门实战：[实战教程第二章2.2：如何快速体验 OceanBase](https://open.oceanbase.com/articles/8600142)
+ 社区版官网-问答：[OceanBase CE 容器下载使用简介](https://open.oceanbase.com/answer/detail?id=508)
+ 教程视频：[2-2-OceanBase Docker 体验.mp4](https://www.bilibili.com/video/BV1dS4y1X7aC?spm_id_from=333.999.0.0)
