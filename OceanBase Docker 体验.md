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

## 启动 OceanBase 数据库实例

+ 根据当前容器部署最大规格的实例

`docker run -p 2881:2881 --name obstandalone -d oceanbase/oceanbase-ce`

+ 部署 mini 的独立实例

`docker run -p 2881:2881 --name obstandalone -e MINI_MODE=1 -d oceanbase/oceanbase-ce`


```sh
[root@obdocker ~]# docker run -itd -p 2881:2881 --name obstandalone -d oceanbase/oceanbase-ce
Unable to find image 'oceanbase/oceanbase-ce:latest' locally
latest: Pulling from oceanbase/oceanbase-ce
13add961a70d: Pull complete
c8175aff0e18: Pull complete
39c994bcc219: Pull complete
71a870d28c6f: Pull complete
Digest: sha256:6c3f458abc38a017e604af1188726174b6dc81f38c96e2869dc2cb04931a8cd8
Status: Downloaded newer image for oceanbase/oceanbase-ce:latest
97f15204c46b431ff429f26effdef406f07ed14c9833665943f20fe9340b8213

# oceanbase/oceanbase-ce = oceanbase/oceanbase-ce:latest，当本地没该镜像时会通过网络从 Docker Hub 拉取。
```

+ 查看运行中的容器

```sh
[root@obdocker ~]# docker ps
CONTAINER ID   IMAGE                    COMMAND              CREATED          STATUS          PORTS                                       NAMES
f38f5dd1570e   oceanbase/oceanbase-ce   "/bin/sh -c _boot"   22 seconds ago   Up 22 seconds   0.0.0.0:2881->2881/tcp, :::2881->2881/tcp   obstandalone
```

## 连接 OceanBase 数据库实例

oceanbase-ce 镜像安装了 OceanBase 数据库客户端 obclient，并提供了默认连接脚本 ob-mysql。

```sh
docker exec -it obstandalone ob-mysql sys # 连接 sys 租户
docker exec -it obstandalone ob-mysql root # 连接用户租户的 root 账户
docker exec -it obstandalone ob-mysql test # 连接用户租户的 test 账户
```

```sh
[root@obdocker ~]# docker exec -it obstandalone ob-mysql sys
login as root@sys
Command is: obclient -h127.1 -uroot@sys -A -Doceanbase -P2881
Welcome to the OceanBase.  Commands end with ; or \g.
Your OceanBase connection id is 3221487727
Server version: 5.7.25 OceanBase 3.1.4 (r10000092022071511-b4bfa011ceaef428782dcb65ae89190c40b78c2f) (Built Jul 15 2022 11:45:14)

Copyright (c) 2000, 2022, OceanBase and/or its affiliates. All rights reserved.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

obclient [oceanbase]>
```

也可以运行本机的 obclient 或者 MySQL 客户端连接实例。

```sh
[root@obdocker ~]# obclient -uroot@sys -h127.0.0.1 -P2881 -c -A oceanbase
Welcome to the OceanBase.  Commands end with ; or \g.
Your OceanBase connection id is 3221487717
Server version: 5.7.25 OceanBase 3.1.4 (r10000092022071511-b4bfa011ceaef428782dcb65ae89190c40b78c2f) (Built Jul 15 2022 11:45:14)

Copyright (c) 2000, 2022, OceanBase and/or its affiliates. All rights reserved.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

obclient [oceanbase]> show databases;
+--------------------+
| Database           |
+--------------------+
| oceanbase          |
| information_schema |
| mysql              |
| SYS                |
| LBACSYS            |
| ORAAUDITOR         |
| test               |
+--------------------+
7 rows in set (0.002 sec)
```


## 使用默认的 test 租户创建业务数据库和表

对 MySQL 模式，使用 MySQL 客户端或 obclient 客户端都可以连接 MySQL 租户。租户的连接帐户格式: **用户名@租户名#集群名**； **集群名:租户名:用户名**。MySQL 租户的管理员用户名默认是 root。

**获取 OceanBase 可用资源:**

```sh
[root@obdocker ~]# obclient -uroot@sys -h127.0.0.1 -P2881 -c -A oceanbase
Welcome to the OceanBase.  Commands end with ; or \g.
Your OceanBase connection id is 3221487749
Server version: 5.7.25 OceanBase 3.1.4 (r10000092022071511-b4bfa011ceaef428782dcb65ae89190c40b78c2f) (Built Jul 15 2022 11:45:14)

Copyright (c) 2000, 2022, OceanBase and/or its affiliates. All rights reserved.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
obclient [oceanbase]> select a.zone,concat(a.svr_ip,':',a.svr_port) observer, cpu_total, cpu_assigned, (cpu_total-cpu_assigned) cpu_free, mem_total/1024/1024/1024 mem_total_gb, mem_assigned/1024/1024/1024 mem_assign_gb, (mem_total-mem_assigned)/1024/1024/1024 mem_free_gb
     from __all_virtual_server_stat a join __all_server b on (a.svr_ip=b.svr_ip and a.svr_port=b.svr_port)
     order by a.zone, a.svr_ip
     ;
+-------+----------------+-----------+--------------+----------+-----------------+-----------------+----------------+
| zone  | observer       | cpu_total | cpu_assigned | cpu_free | mem_total_gb    | mem_assign_gb   | mem_free_gb    |
+-------+----------------+-----------+--------------+----------+-----------------+-----------------+----------------+
| zone1 | 127.0.0.1:2882 |        16 |         13.5 |      2.5 | 13.000000000000 | 12.350000000559 | 0.649999999440 |
+-------+----------------+-----------+--------------+----------+-----------------+-----------------+----------------+
1 row in set (0.003 sec)
```


**连接 test 租户创建数据库**

```sql
[root@obdocker ~]# obclient -uroot@test -h127.0.0.1 -P2881 -c -A oceanbase

obclient [oceanbase]> create database my_1st_db;
Query OK, 1 row affected (0.015 sec)

obclient [oceanbase]> show databases;
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
[root@obdocker ~]#  obclient -h127.0.0.1 -umy_1st_user@test -P2881 -ppassword -c -A my_1st_db;
Welcome to the OceanBase.  Commands end with ; or \g.
Your OceanBase connection id is 3221487697
Server version: 5.7.25 OceanBase 3.1.4 (r10000092022071511-b4bfa011ceaef428782dcb65ae89190c40b78c2f) (Built Jul 15 2022 11:45:14)

Copyright (c) 2000, 2022, OceanBase and/or its affiliates. All rights reserved.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

obclient [my_1st_db]> create table my_1st_table (id int, name varchar(10));
Query OK, 0 rows affected (0.055 sec)

obclient [my_1st_db]> desc my_1st_table;
+-------+-------------+------+-----+---------+-------+
| Field | Type        | Null | Key | Default | Extra |
+-------+-------------+------+-----+---------+-------+
| id    | int(11)     | YES  |     | NULL    |       |
| name  | varchar(10) | YES  |     | NULL    |       |
+-------+-------------+------+-----+---------+-------+
2 rows in set (0.001 sec)
```


**插入业务数据:**

```sql
obclient [my_1st_db]> insert into my_1st_table values (1, 'hi,ob');
Query OK, 1 row affected (0.008 sec)

obclient [my_1st_db]>  select * from my_1st_db.my_1st_table;
+------+-------+
| id   | name  |
+------+-------+
|    1 | hi,ob |
+------+-------+
1 row in set (0.001 sec)
```


Ref:

+ [Linux 安装 Docker](https://docs.docker.com/engine/install/centos/)
+ [OceanBase GitHub](https://github.com/oceanbase/oceanbase/wiki/Deploy-OceanBase-with-docker)
