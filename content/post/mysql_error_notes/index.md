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

经查阅，也是版本问题，`change master to`已经在MySql8.4中被`CHANGE REPLICATION SOURCE TO`取代，https://dev.mysql.com/doc/refman/8.4/en/replication-howto-slaveinit.html。