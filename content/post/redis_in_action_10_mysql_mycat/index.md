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

mysql> insert into student(name,age,score) values('farb',26,96);
Query OK, 1 row affected (0.01 sec)
```

## 创建java项目，用java读写mysql集群和redis
> 向主数据库写数据，这些数据会自动同步到从数据库。
> 读数据时，先从redis中读，可以提升性能。如果redis中没有，再从从数据库中读，最后再写redis。
> 读写分离方式也可以提升性能。

1. 确保pom.xml中添加了redis和mysql的依赖

```xml
        <dependency>
            <groupId>redis.clients</groupId>
            <artifactId>jedis</artifactId>
            <version>3.3.0</version>
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>8.0.33</version>
        </dependency>
```

2. 高并发的场景里，至少会用到MySql的主从复制集群，无论是性能还是可用性都比单机版MySql好。在此基础上再引入Redis作为缓存服务器，能进一步提升数据库服务的性能。
![](https://s3.bmp.ovh/imgs/2024/08/27/5b1966ecfb31229c.png)

3. Java代码示例
```java
public class MySqlClusterDemo {
    PreparedStatement masterPreparedStatement;
    PreparedStatement slavePreparedStatement;
    private Jedis jedis;
    private Connection masterConnection;
    private Connection slaveConnection;

    /**
     * 程序入口
     * @param args 命令行参数
     */
    public static void main(String[] args) {
        // 创建示例对象
        MySqlClusterDemo demo = new MySqlClusterDemo();
        // 初始化数据库连接
        demo.init();
        // 插入示例数据
        demo.insert();
        // 场景1：从数据库查询ID为1的名称并打印
        System.out.println(demo.getNameById(1));
        // 场景2：从redis缓存再次查询并打印，用于验证数据一致性
        System.out.println(demo.getNameById(1));
    }

    /**
     * 初始化数据库和Redis连接
     * 该方法在类初始化时调用，负责建立到MySQL主从数据库的连接以及到Redis服务器的连接
     */
    private void init() {
        // MySQL主库连接URL
        String mysqlMasterUrl = "jdbc:mysql://127.0.0.1:3306/redisDemo";
        // MySQL从库连接URL
        String mysqlSlaveUrl = "jdbc:mysql://127.0.0.1:3316/redisDemo";
        // 数据库用户名
        String user = "root";
        // 数据库密码
        String password = "123456";
        try {
            // 建立到MySQL主库的连接
            masterConnection = DriverManager.getConnection(mysqlMasterUrl, user, password);
            // 建立到MySQL从库的连接
            slaveConnection = DriverManager.getConnection(mysqlSlaveUrl, user, password);
        } catch (SQLException e) {
            // 如果发生SQL异常，则抛出运行时异常
            throw new RuntimeException(e);
        }
        // 建立到Redis服务器的连接
        jedis = new Jedis("127.0.0.1", 6379);
    }

    /**
     * 插入学生数据的方法
     * 该方法用于向学生表中插入一条新的学生记录，包括姓名、年龄和分数
     * 由于数据库操作可能存在异常，故在此方法中进行了异常捕获和处理
     */
    private void insert() {
        try {
            // 准备插入语句，将预设的学生数据插入到主数据库中
            masterPreparedStatement = masterConnection.prepareStatement("insert into student(name,age,score) values ('Frank',18,95)");
            // 执行插入操作
            masterPreparedStatement.executeUpdate();
        } catch (SQLException e) {
            // 捕获SQL异常，避免程序因未处理的异常而中断
            throw new RuntimeException(e);
        }
    }


    /**
     * 根据学生ID获取学生姓名
     * 首先检查Redis中是否存在该数据，如果存在，则直接返回，以提高查询效率
     * 如果Redis中不存在，则从数据库中查询，并将结果存储到Redis中，以便后续查询
     *
     * @param id 学生ID
     * @return 学生姓名，如果找不到则返回空字符串
     */
    private String getNameById(int id) {
        // 构造Redis的键
        String key = "student:" + id;
        // 初始化姓名为空字符串
        String name = "";
        // 检查Redis中是否存在该键
        if (jedis.exists(key)) {
            // 如果存在，则打印消息并从Redis中获取姓名
            System.out.println("从redis中获取数据:id=" + id + "存在于数据库");
            name = jedis.get(key);
            // 打印获取到的姓名
            System.out.println("Name is " + name);
            // 返回获取到的姓名
            return name;
        }
        // 尝试从数据库中获取姓名
        try {
            // 准备数据库查询语句
            slavePreparedStatement = slaveConnection.prepareStatement("select name from student where id = ?");
            // 设置查询参数为学生ID
            slavePreparedStatement.setInt(1, id);
            // 执行查询并获取结果集
            ResultSet resultSet = slavePreparedStatement.executeQuery();
            // 如果结果集中有数据
            if (resultSet.next()) {
                // 获取姓名
                name = resultSet.getString("name");
                // 打印从数据库中获取数据的消息
                System.out.println("从数据库中获取数据:id=" + id + "存在于数据库");
            }
            // 将姓名存储到Redis中,并设置300s过期时间，如果没有找到数据，缓存为空值，防止缓存穿透
            SetParams params = SetParams.setParams().ex(300);
            jedis.set(key, name, params);
            // 返回获取到的姓名
            return name;
        } catch (SQLException e) {
            // 如果发生SQL异常，则抛出运行时异常
            throw new RuntimeException(e);
        }
    }

}

```
## mysql主从集群整合redis主从集群
1. 搭建一主一从的mysql集群，一主一从的Redis集群，如下图：
![](https://s3.bmp.ovh/imgs/2024/08/28/f8fe6efa299a0b80.png)

2. 主从模式的MySQL集群上面已经配置，接下来重新复习一下配置一主一从的Redis集群，最终配置一览表如下：
| docker容器名  | IP地址和端口   | 说明        |
| ------------- | -------------- | ----------- |
| myMasterMysql | 127.0.0.1:3306 | MySQL主节点 |
| mySlaveMysql  | 127.0.0.1:3316 | MySQL从节点 |
| redis-master  | 127.0.0.1:6379 | Redis主节点 |
| redis-slave   | 127.0.0.1:6380 | Redis从节点 |

```sh
# 启动redis主服务器容器
PS D:\code\blogs\farb.github.io> docker run -itd --name redis-master -p 6379:6379 redis
7766df2403d2da7852380710245441da684a5aa19b040995dbd1606e78b6855f

# 启动redis从服务器容器
PS D:\code\blogs\farb.github.io> docker run -itd --name redis-slave -p 6380:6379 redis 
9ee23465b5a829d440821014d30998b2b4d76cd10e10d0a01bcfd8081bd5d626

# 获取redis主服务器的IP地址
PS D:\code\blogs\farb.github.io>  docker inspect -f "{{.NetworkSettings.IPAddress}}" redis-master
172.17.0.4

# 配置redis从服务器为从节点
PS D:\code\blogs\farb.github.io> docker exec -it redis-slave bash
root@9ee23465b5a8:/data# redis-cli -h localhost -p 6379
# 这里为了方便，通过命令slaveof设置redis从服务器为从节点
localhost:6379> slaveof 172.17.0.4 6379
OK

# 查看主从关系状态正常
localhost:6379> info replication
# Replication
role:slave
master_host:172.17.0.4
master_port:6379
master_link_status:up
```

3. java代码改进点：
   1. 读取数据时，优先从“从redis缓存”中读取；
   2. 从数据库查询到数据后，将数据写入“主Redis缓存”中，通过redis的主从复制自动同步到从Redis中；

```java
public class MySqlClusterDemoV2 {
    // 主mysql预编译sql语句对象
    PreparedStatement masterPreparedStatement;
    // 从mysql预编译sql语句对象
    PreparedStatement slavePreparedStatement;
    // 主Redis客户端对象
    private Jedis masterJedis;
    // 从Redis客户端对象
    private Jedis slaveJedis;
    // 主数据库连接对象
    private Connection masterConnection;
    // 从数据库连接对象
    private Connection slaveConnection;

    /**
     * 程序入口
     *
     * @param args 命令行参数
     */
    public static void main(String[] args) {
        // 创建示例对象
        MySqlClusterDemoV2 demo = new MySqlClusterDemoV2();
        // 初始化数据库连接
        demo.init();
        // 插入示例数据
        demo.insert();
        // 场景1：从数据库查询ID为1的名称并打印
        System.out.println(demo.getNameById(1));
        // 场景2：从redis缓存再次查询并打印，用于验证数据一致性
        System.out.println(demo.getNameById(1));
    }

    /**
     * 初始化数据库和Redis连接
     * 该方法在类初始化时调用，负责建立到MySQL主从数据库的连接以及到Redis服务器的连接
     */
    private void init() {
        // MySQL主库连接URL
        String mysqlMasterUrl = "jdbc:mysql://127.0.0.1:3306/redisDemo";
        // MySQL从库连接URL
        String mysqlSlaveUrl = "jdbc:mysql://127.0.0.1:3316/redisDemo";
        // 数据库用户名
        String user = "root";
        // 数据库密码
        String password = "123456";
        try {
            // 建立到MySQL主库的连接
            masterConnection = DriverManager.getConnection(mysqlMasterUrl, user, password);
            // 建立到MySQL从库的连接
            slaveConnection = DriverManager.getConnection(mysqlSlaveUrl, user, password);
        } catch (SQLException e) {
            // 如果发生SQL异常，则抛出运行时异常
            throw new RuntimeException(e);
        }
        // 建立到主Redis服务器的连接
        masterJedis = new Jedis("127.0.0.1", 6379);
        // 建立到从Redis服务器的连接
        slaveJedis = new Jedis("127.0.0.1", 6380);
    }

    /**
     * 插入学生数据的方法
     * 该方法用于向学生表中插入一条新的学生记录，包括姓名、年龄和分数
     * 由于数据库操作可能存在异常，故在此方法中进行了异常捕获和处理
     */
    private void insert() {
        try {
            // 准备插入语句，将预设的学生数据插入到主数据库中
            masterPreparedStatement = masterConnection.prepareStatement("insert into student(name,age,score) values ('Frank',18,95)");
            // 执行插入操作
            masterPreparedStatement.executeUpdate();
        } catch (SQLException e) {
            // 捕获SQL异常，避免程序因未处理的异常而中断
            throw new RuntimeException(e);
        }
    }


    /**
     * 根据学生ID获取学生姓名
     * 首先检查Redis中是否存在该数据，如果存在，则直接返回，以提高查询效率
     * 如果Redis中不存在，则从数据库中查询，并将结果存储到Redis中，以便后续查询
     *
     * @param id 学生ID
     * @return 学生姓名，如果找不到则返回空字符串
     */
    private String getNameById(int id) {
        // 构造Redis的键
        String key = "student:" + id;
        // 初始化姓名为空字符串
        String name = "";
        // 检查Redis中是否存在该键
        if (slaveJedis.exists(key)) {
            // 如果存在，则打印消息并从Redis中获取姓名
            System.out.println("从redis中获取数据:id=" + id + "存在于数据库");
            name = slaveJedis.get(key);
            // 打印获取到的姓名
            System.out.println("Name is " + name);
            // 返回获取到的姓名
            return name;
        }
        // 尝试从数据库中获取姓名
        try {
            // 准备数据库查询语句
            slavePreparedStatement = slaveConnection.prepareStatement("select name from student where id = ?");
            // 设置查询参数为学生ID
            slavePreparedStatement.setInt(1, id);
            // 执行查询并获取结果集
            ResultSet resultSet = slavePreparedStatement.executeQuery();
            // 如果结果集中有数据
            if (resultSet.next()) {
                // 获取姓名
                name = resultSet.getString("name");
                // 打印从数据库中获取数据的消息
                System.out.println("从数据库中获取数据:id=" + id + "存在于数据库");
            }
            // 将姓名存储到Redis中,并设置300s过期时间，如果没有找到数据，缓存为空值，防止缓存穿透
            SetParams params = SetParams.setParams().ex(300);
            masterJedis.set(key, name, params);
            // 返回获取到的姓名
            return name;
        } catch (SQLException e) {
            // 如果发生SQL异常，则抛出运行时异常
            throw new RuntimeException(e);
        }
    }

}

```
----
**MyCAT不维护了，建议不要使用了，不过可以看下它的原理。建议直接使用分布式数据库TiDB** 
----
# Redis整合MySql和MyCat分库组件
> MyCAT是一个开源的分布式数据库组件，一般用它实现对数据库的分库分表功能，从而提升对数据库（尤其是大数据库表）的访问性能。
> [MyCAT官方文档](http://www.mycat.org.cn/)

## 分库分表概述
假设一电商系统业务量非常大，流水表（假设主键id）可能达到“亿”级规模，甚至更大。如果需要查询数据，即使使用索引等数据库优化的手段，也会成为性能上的瓶颈。此时可以考虑如下思路。

1. 在不同的10个数据库中，同时创建10张流水表，这些表的结构完全一致。
2. 在1号数据库中只存放id%10=1的数据，比如存放id=1、11、21等的数据，2号数据库存放id%10=2的数据，以此类推。

这样就将一个大的流水表分成10张子表，而MyCAT组件能把应用程序对流水表的请求分散到这10张子表上，实际业务中子表的个数可以根据业务需求来定。由于把大表的数据分散到若干子表中，因此每次请求所面对的数据总量有效降低，从而能感受到“分表”对提升数据库访问性能的帮助。实际项目中，尽量将子表分散创建到不同的机器上，这样子表之间就具有更高的并发能力，不推荐在同一台机器上的同一个数据库进行分表操作。

基于成本的考虑，或许无法为每个子表分配一台机器，但可以将不同的子表分散地创建在同一主机不同的数据库上，因而得出下面结论。

**子表创建在不同的机器上的性能 > 子表创建在同一台机器上的不同数据库的性能 > 子表创建在同一台机器上的同一个数据库的性能**

## 用MyCAT组件实现分库分表
> MyCAT组件默认工作在8066端口，Java程序不是直接和MySQL等数据库直连，而是和MyCAT组件交互，再由MyCAT组件和数据库交互，MyCAT根据配置好的分库分表规则把请求发送到对应的数据库上，得到请求再返回给应用程序。

![](https://s3.bmp.ovh/imgs/2024/08/29/8108fcf768f00f2f.png)

为了实现分库分表效果，一般需要配置MyCAT组件的三个文件。

| 文件名     | 说明                                                                             |
| ---------- | -------------------------------------------------------------------------------- |
| server.xml | MyCAT组件的配置文件，一般用于配置MyCAT组件的端口号、用户名和密码、编码、字符集等 |
| schema.xml | MyCAT组件的分库分表配置文件，一般用于配置各分库的连接信息                        |
| rule.xml   | MyCAT规则配置文件，一般用于配置分库分表规则                                      |

1. server.xml配置如下：
**配置了MyCAT组件的工作端口8066和管理端口9066，配置连接到MyCAT组件的用户名和密码，这里是MyCAT的用户名密码和数据库，而不是MySQL的，实践过程中一般和MySQL保持一致。**
``` xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mycat:server SYSTEM "server.dtd">
<mycat:server xmlns:mycat="http://io.mycat/">
    <system>
        <property name="serverPort">8066</property>
        <property name="managePort">9066</property>
    </system>
    <user name="root">
        <property name="password">123456</property>
        <property name="schemas">redisDemo</property>
    </user>
</mycat:server>
```

2. schema.xml配置如下：
**定义了redisDemo数据库的student表，按照mod-long规则分布到dn1,dn2,dn3这3个数据节点上,然后给出了dn1,dn2,dn3这3个数据节点的定义，分别指向host1,host2,host3这3个MySQL数据库。随后又定义了host1,host2,host3。**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mycat:schema SYSTEM "schema.dtd">
<mycat:schema xmlns:mycat="http://io.mycat/">
    <schema name="redisDemo">
        <table name="student" dataNode="dn1,dn2,dn3" rule="mod-long" />
    </schema>
    <dataNode name="dn1" dataHost="host1" database="redisDemo" />
    <dataNode name="dn2" dataHost="host2" database="redisDemo" />
    <dataNode name="dn3" dataHost="host3" database="redisDemo" />
    <dataHost name="host1" maxCon="100" minCon="10" balance="0" writeType="0" dbType="mysql"
        dbDriver="native">
        <heartbeat>select user()</heartbeat>
        <writeHost host="hostM1" url="172.17.0.2:3306" user="root" password="123456" />
    </dataHost>
    <dataHost name="host2" maxCon="100" minCon="10" balance="0" writeType="0" dbType="mysql"
        dbDriver="native">
        <heartbeat>select user()</heartbeat>
        <writeHost host="hostM2" url="172.17.0.3:3306" user="root" password="123456" />
    </dataHost>
    <dataHost name="host3" maxCon="100" minCon="10" balance="0" writeType="0" dbType="mysql"
        dbDriver="native">
        <heartbeat>select user()</heartbeat>
        <writeHost host="hostM3" url="172.17.0.4:3306" user="root" password="123456" />
    </dataHost>
</mycat:schema>
```

3. rule.xml配置如下：
**定义mod-long规则，用于将id字段按照模3规则分配到dn1,dn2,dn3这3个数据节点上。**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mycat:rule SYSTEM "rule.dtd">
<mycat:rule xmlns:mycat="http://io.mycat/">
    <tableRule name="mod-long">
        <rule>
            <columns>id</columns>
            <algorithm>mod-long</algorithm>
        </rule>
    </tableRule>
    <function name="mod-long" class="io.mycat.route.function.PartitionByMod">
        <property name="count" value="3" />
    </function>
</mycat:rule>
```

**综上，给出如下针对分库分表相关动作的定义：**
> 1. 应用程序如果要使用MyCAT,就需要用户名root和密码123456连接到MyCAT组件。
> 2. 假设要插入id为1的student的数据，根据schema.xml配置，会先根据mod-long规则对id进行模3处理，结果是1，所以会插入到host2所定义的172.17.0.3:3306的数据库的student表，如果要进行读取、删除和修改，就会先对id模3，再把请求发送到对应的数据库上。

## Java、MySQL和MyCAT组件的整合
**使用上面的配置文件，继续实践**
1. 通过三个命令，准备三个包含mysql的docke容器：
```bash
PS D:\code\blogs\farb.github.io> docker run -itd -p 3306:3306 --name mysqlHost1 -e MYSQL_ROOT_PASSWORD=123456 mysql
8f4d6474e3e45aca4d037b696a8a0d3bf98c620e3222681c06c4f0e5951c8a4a
PS D:\code\blogs\farb.github.io> docker run -itd -p 3316:3306 --name mysqlHost2 -e MYSQL_ROOT_PASSWORD=123456 mysql 
e0b0a0691740cc8f6959ff16f0cc54306519fe63c32b6393d33a0d999b19cb41
PS D:\code\blogs\farb.github.io> docker run -itd -p 3326:3306 --name mysqlHost3 -e MYSQL_ROOT_PASSWORD=123456 mysql
f18e144b62b4897d7fd61ec6aa3d3d532535ea61917c2d1a7bdaea034f48d275

# 创建完成后，查看每个容器对应的IP
PS D:\code\blogs\farb.github.io> docker inspect -f "{{.NetworkSettings.IPAddress}}" mysqlHost1
172.17.0.2
PS D:\code\blogs\farb.github.io> docker inspect -f "{{.NetworkSettings.IPAddress}}" mysqlHost2
172.17.0.3
PS D:\code\blogs\farb.github.io> docker inspect -f "{{.NetworkSettings.IPAddress}}" mysqlHost3
172.17.0.4
```
**整理成表格，方便查阅**
| 容器名 | IP  |
| -------- | -------- |
| mysqlHost1 | 172.17.0.2 |
| mysqlHost2 | 172.17.0.3 |
| mysqlHost3 | 172.17.0.4 |

2. 分别进入三个容器，建库建表：
```sh
PS D:\code\blogs\farb.github.io> docker exec -it mysqlHost1 bash
bash-5.1# mysql -u root -p
Enter password:
mysql> create database redisDemo;
Query OK, 1 row affected (0.01 sec)

mysql> use redisDemo;
Database changed
mysql> create table student (
    -> id int not null primary key,
    -> name varchar(20),
    -> age int,
    -> score float
    -> );
Query OK, 0 rows affected (0.01 sec)

# mysqlHost2和mysqlHost3执行相同的操作，此处略
# docker exec -it mysqlHost2 bash
# docker exec -it mysqlHost3 bash
```

3. 通过docker run命令启动一个MyCAT组件的容器（我发现docker仓库上没有可用的mycat镜像，于是自己根据源码生成的镜像，可以参考这篇文章[MyCAT源码构建镜像](https://blog.csdn.net/weixin_43431218/article/details/133079200)）
```bash
PS D:\code\blogs\farb.github.io> docker run -itd --name mycat -p 8066:8066 -p 9066:9066 -v D:\ArchitectPracticer\Redis\MyCAT\server.xml:/usr/local/mycat/conf/server.xml:ro -v D:\ArchitectPracticer\Redis\MyCAT\schema.xml:/usr/local/mycat/conf/schema.xml:ro -v D:\ArchitectPracticer\Redis\MyCAT\rule.xml:/usr/local/mycat/conf/rule.xml:ro mycat:1.6.7.6
```
