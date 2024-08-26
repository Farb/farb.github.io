---
title: 基于Docker的Redis实战-Redis整合Mysql集群和MyCat分库分表组件
description:
slug: redis_in_action_10_mysql_mycat
date: 2024-08-21
image: 
categories:
    - Redis
tags:
    - redis
    - docker
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---
源码链接： https://gitee.com/farb/architect-practicer-code 

# Redis整合MySql主从集群
## 用docker搭建MySql一主一从集群
1. 在主Mysql服务器的操作会自动同步到从Mysql服务器，比如插入数据，删除数据，更新数据和新建数据库等都会同步到从服务器，通过这种同步的动作，主从服务器之间可以保持数据一致性。
2. 一般项目都是向主服务器写数据，从“从服务器”读数据，这种读写分离的方式可以提升数据库的性能。
   
```sh
# 拉取镜像
PS D:\code\blogs\farb.github.io> docker pull mysql

# 本机中新建mysql的配置文件 D:\ArchitectPracticer\Redis\MySql\MasterMySql\conf\my.cnf ，内容如下：
[mysqld]
pid-file=/var/run/mysqld/mysqld.pid
socket=/var/run/mysqld/mysqld.sock
datadir=/var/lib/mysql
server-id=1 # 主服务器的id，这个id不能重复
log-bin=mysql-master-bin

# 启动主服务器，设置环境变量，MYSQL_ROOT_PASSWORD为root用户的密码，同时挂载配置文件目录和数据目录
PS D:\code\blogs\farb.github.io> docker run -itd --name myMasterMysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 -v D:\ArchitectPracticer\Redis\MySql\MasterMySql\conf:/etc/mysql/conf.d -v D:\ArchitectPracticer\Redis\MySql\MasterMySql\data:/var/lib/mysql mysql 
7b1067615f0d6a663d38d1191544d3d4d394bc5768bb37a25109b7073cd5e06b

# 查看运行的容器
PS D:\code\blogs\farb.github.io> docker ps
CONTAINER ID   IMAGE     COMMAND                   CREATED              STATUS              PORTS                               NAMES
7b1067615f0d   mysql     "docker-entrypoint.s…"   About a minute ago   Up About a minute   0.0.0.0:3306->3306/tcp, 33060/tcp   myMasterMysql

# 获取主服务器的IP地址
D:\code\blogs\farb.github.io> docker inspect myMasterMysql -f "{{.NetworkSettings.IPAddress}}" 
172.17.0.2

# 登录主服务器
PS D:\code\blogs\farb.github.io> docker exec -it myMasterMysql bash
bash-5.1# mysql -u root -p
# 这句告警说明配置文件没有生效，需要手动在容器中修改权限， bash-5.1# chmod 644 /etc/mysql/conf.d/my.cnf
mysql: [Warning] World-writable config file '/etc/mysql/conf.d/my.cnf' is ignored.
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 9
Server version: 8.4.0 MySQL Community Server - GPL

Copyright (c) 2000, 2024, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

# 查看主服务器的bin log, 主要关注前2列，主从集群同步所用到的日志文件名和当前同步的位置。
# 注意：show master status;已经被废弃，使用SHOW BINARY LOG STATUS;代替
mysql> SHOW BINARY LOG STATUS;
+-------------------------+----------+--------------+------------------+-------------------+
| File                    | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+-------------------------+----------+--------------+------------------+-------------------+
| mysql-master-bin.000002 |      158 |              |                  |                   |
+-------------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)

```

新开一个命令行窗口继续设置从服务器：

```sh
# mysql从服务器的配置文件如下
[mysqld]
pid-file=/var/run/mysqld/mysqld.pid
socket=/var/run/mysqld/mysqld.sock
datadir=/var/lib/mysql
server-id=2 # 从服务器的id，这个id不能和主服务器的一致
log-bin=mysql-slave-bin # 二进制文件的名字
```

```sh
# 启动从服务器
PS D:\code\blogs\farb.github.io> docker run -itd --name mySlaveMysql -p 3316:3306 -e MYSQL_ROOT_PASSWORD=123456 -v D:\ArchitectPracticer\Redis\MySql\SlaveMySql\conf:/etc/mysql/conf.d -v D:\ArchitectPracticer\Redis\MySql\SlaveMySql\data:/var/lib/mysql mysql 

PS D:\code\blogs\farb.github.io> docker exec -it mySlaveMysql /bin/bash

# 确认可以在从服务器的容器中连接到主服务器
bash-5.1# mysql -h 172.17.0.2 -u root -p
mysql: [Warning] World-writable config file '/etc/mysql/conf.d/my.cnf' is ignored.
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 18
Server version: 8.4.0 MySQL Community Server - GPL

Copyright (c) 2000, 2024, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

#  exit退出主服务器的连接
mysql> exit
Bye

# 连接到从服务器
bash-5.1# mysql -h localhost -u root -p
mysql: [Warning] World-writable config file '/etc/mysql/conf.d/my.cnf' is ignored.
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 11
Server version: 8.4.0 MySQL Community Server - GPL

Copyright (c) 2000, 2024, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

# 配置主从关系，这里注意`change master to`已经在MySql8中被`CHANGE REPLICATION SOURCE TO`取代。每个字段的意思很明显，主要是`SOURCE_log_pos`和`SOURCE_log_file`要和上面`show binary log status;`的结果一致。`
mysql> CHANGE REPLICATION SOURCE TO SOURCE_host='172.17.0.2',SOURCE_port=3306,SOURCE_user='root',SOURCE_password='123456',SOURCE_log_pos=158,SOURCE_log_file='mysql-master-bin.000002';
Query OK, 0 rows affected, 2 warnings (0.04 sec)

# start slave;也变成了start replica; 详见 https://dev.mysql.com/doc/refman/8.4/en/start-replica.html
mysql> start replica;
Query OK, 0 rows affected (0.01 sec)

# 启动主从复制
mysql> start replica;
Query OK, 0 rows affected (0.01 sec)

# 查看主从复制状态，命令结尾符\G是以垂直列表的形式显示。看到Replica_IO_Running: Yes 和 Replica_SQL_Running: Yes，说明配置主从复制成功
mysql> show replica status\G;
*************************** 1. row ***************************
             Replica_IO_State: Waiting for source to send event
                  Source_Host: 172.17.0.2
                  Source_User: root
                  Source_Port: 3306
                Connect_Retry: 60
              Source_Log_File: mysql-master-bin.000002
          Read_Source_Log_Pos: 158
               Relay_Log_File: 0f827dad7de8-relay-bin.000003
                Relay_Log_Pos: 335
        Relay_Source_Log_File: mysql-master-bin.000002
           Replica_IO_Running: Yes
          Replica_SQL_Running: Yes
... 省略一部分输出
1 row in set (0.00 sec)

ERROR:
No query specified


# 回到主服务器，创建数据库
mysql> create database MasterSlaveDemo;
Query OK, 1 row affected (0.00 sec)

# 可以看到主服务器中有新建的数据库MasterSlaveDemo
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| MasterSlaveDemo    |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0.00 sec)

# 再去从服务器中查看，检查是否同步成功，结果发现同步成功
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| MasterSlaveDemo    |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0.00 sec)
```

## 准备数据
```sh
# 创建数据库
mysql> create database redisDemo;
Query OK, 1 row affected (0.01 sec)

# 切换到数据库
mysql> use redisDemo;
Database changed

# 创建表
mysql> create table student (
    -> id int not null primary key auto_increment,
    -> name varchar(20),
    -> age int,
    -> score float
    -> );
Query OK, 0 rows affected (0.02 sec)

# 插入数据
mysql> insert into student (name,age,score) values('Peter',18,100);
Query OK, 1 row affected (0.01 sec)

mysql> insert into student (name,age,score) values('Tom',17,98),('John',17,99);
Query OK, 2 rows affected (0.01 sec)
Records: 2  Duplicates: 0  Warnings: 0
```

## 创建java项目，用java读写mysql集群和redis

## mysql主从集群整合redis主从集群

# Redis整合MySql和MyCat分库组件
