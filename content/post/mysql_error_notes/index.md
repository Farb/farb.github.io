---
title: MySql报错记录集
description:
slug: mysql_error_notes
date: 2024-08-21
image: 
categories:
    - MySql
tags:
    - MySql
    - error_notes
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---

### 1. [Warning] World-writable config file '/etc/mysql/conf.d/my.cnf' is ignored.
**警告任何用户都可以修改配置文件，太不安全，所以Mysql把这个配置文件忽略了。 文件权限太高，需要降低文件权限。本错误是在docker容器中出现的，所以需要进入docke容器中执行以下命令。**

```sh
PS D:\code\blogs\farb.github.io> docker exec -it myMasterMysql bash
bash-5.1# chmod 644 /etc/mysql/conf.d/my.cnf
```

# 2. mysql> show master status; 命令报错语法错误
报错如下：

ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near 'master status' at line 1

困扰了很久，最终找到了原因，是mysql版本问题，我使用的是mysql 8.4，`show master status;`已经被废弃，使用`SHOW BINARY LOG STATUS;`代替
具体查看官方说明，https://dev.mysql.com/doc/refman/8.4/en/show-master-status.html

# 3. mysql> change master to master_host='172.17.0.2',master_port=3306,master_user='root',master_password='123456',master_log_pos=158,master_log_file='mysql-master-bin.000002'; 设置主从关系时报语法错误
错误提示如下：

ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near 'master to master_host='172.17.0.2',master_port=3306,master_user='root',master_pa' at line 1

经查阅，也是版本问题，`change master to`已经在MySql8中被`CHANGE REPLICATION SOURCE TO`取代，https://dev.mysql.com/doc/refman/8.4/en/replication-howto-slaveinit.html。

# 4. (HTTP code 500) server error - Ports are not available: exposing port TCP 0.0.0.0:3316 -> 0.0.0.0:0: listen tcp 0.0.0.0:3316: bind: An attempt was made to access a socket in a way forbidden by its access permissions.

# 5. Public Key Retrieval is not allowed
**使用数据库客户端DBeaver连接mysql时，报错如下：`Public Key Retrieval is not allowed`。**

“Public Key Retrieval is not allowed” 错误是由于 MySQL 连接驱动程序的默认行为更改所引起的。在 MySQL 8.0 版本及更新版本中，默认情况下禁用了通过公钥检索用户密码的功能。

在旧版本的 MySQL 中，客户端连接到服务器时，可以使用公钥来检索用户密码。这种机制称为 “public key retrieval”，它允许客户端使用公钥来解密在服务器端加密的密码。

然而，为了提高安全性，MySQL 开发团队在较新的版本中禁用了这个功能。禁用公钥检索可以防止恶意用户通过获取公钥来获取用户密码。相反，客户端必须使用其他安全的方法来进行身份验证，例如使用预共享密钥或使用 SSL/TLS 连接。

解决方式1. 修改DBeaver中的驱动属性的参数为`allowPublicKeyRetrieval=true` 参考https://blog.csdn.net/qq_33472553/article/details/139107958：。但是这种方式会导致主从模式的数据库集群同步失败，报错见第6点，也即是认证需要安全连接。

# 6. 从数据库没有同步主数据库的数据，通过`show replica status\G`发现报错： Last_IO_Error: Error connecting to source 'root@172.17.0.2:3306'. This was attempt 10/10, with a delay of 60 seconds between attempts. Message: Authentication plugin 'caching_sha2_password' reported error: Authentication requires secure connection.
```sh
# 停止从节点
mysql> stop replica;
Query OK, 0 rows affected (0.00 sec)

# 重置从库配置
mysql> reset replica all;
Query OK, 0 rows affected (0.02 sec)

# 重新配置主从配置关系，主要是最后一句GET_SOURCE_PUBLIC_KEY=1
mysql> CHANGE REPLICATION SOURCE TO SOURCE_host='172.17.0.2',SOURCE_port=3306,SOURCE_user='root',SOURCE_password='123456',SOURCE_log_pos=158,SOURCE_log_file='mysql-master-bin.000004',GET_SOURCE_PUBLIC_KEY=1;
Query OK, 0 rows affected, 2 warnings (0.04 sec)

# 启动从节点
mysql> start replica;
Query OK, 0 rows affected (0.01 sec)

# 再次查看从节点状态，发现没有错误了
mysql> show replica status\G;
*************************** 1. row ***************************
             Replica_IO_State: Waiting for source to send event
                  Source_Host: 172.17.0.2
                  Source_User: root
                  Source_Port: 3306
                Connect_Retry: 60
              Source_Log_File: mysql-master-bin.000004
          Read_Source_Log_Pos: 158
               Relay_Log_File: 0f827dad7de8-relay-bin.000002
                Relay_Log_Pos: 335
        Relay_Source_Log_File: mysql-master-bin.000004
           Replica_IO_Running: Yes
          Replica_SQL_Running: Yes
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Source_Log_Pos: 158
              Relay_Log_Space: 553
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Source_SSL_Allowed: No
        Seconds_Behind_Source: 0
Source_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
             Source_Server_Id: 1
                  Source_UUID: 8e3f2ec2-5fbf-11ef-a070-0242ac110002
             Source_Info_File: mysql.slave_master_info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
    Replica_SQL_Running_State: Replica has read all relay log; waiting for more updates
           Source_Retry_Count: 10
```

# 7. MySql主从集群，从服务器同步报错: Coordinator stopped because there were error(s) in the worker(s). The most recent failure being: Worker 1 failed executing transaction 'ANONYMOUS' at source log mysql-master-bin.000004, end_log_pos 35
**当前我的分析及解决步骤如下：**
1. 查看从节点的日志文件，发现报错信息如下：
```log
2024-08-27 22:46:06 2024-08-27T14:46:06.499064Z 7 [ERROR] [MY-010584] [Repl] Replica SQL for channel '': Worker 1 failed executing transaction 'ANONYMOUS' at source log mysql-master-bin.000004, end_log_pos 354; Error 'Can't drop database 'redisdemo'; database doesn't exist' on query. Default database: 'redisdemo'. Query: 'drop database redisDemo', Error_code: MY-001008
2024-08-27 22:46:06 2024-08-27T14:46:06.499234Z 6 [ERROR] [MY-010586] [Repl] Error running query, replica SQL thread aborted. Fix the problem, and restart the replica SQL thread with "START REPLICA". We stopped at log 'mysql-master-bin.000004' position 158
```
2. 观察到报错信息，发现报错信息提示，删除数据库数据库不存在。因为第一次实践时，创建数据库redisDemo时没有同步成功，所以在主节点执行了`drop database redisDemo`。
3. 为了保持数据操作同步，需要将之前创建的语句手动在从节点执行一下`create database redisDemo;`。
4. 创建数据库之后，`stop replica;`关闭主从同步，然后再`start replica;`开启主从同步，即可解决从节点同步报错的问题。`show replica status\G;`查看从节点正常。

# 8. ERROR 1872 (HY000): Replica failed to initialize applier metadata structure from the repository

解决办法： 参考：https://blog.csdn.net/m0_66011019/article/details/136429224

```bash
mysql> reset replica;
Query OK, 0 rows affected (0.04 sec)
```

