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
# Redis整合MySql和MyCat分库组件
