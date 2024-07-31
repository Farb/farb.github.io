---
title: 基于Docker的Redis实战--搭建Redis集群
description:
slug: redis_in_action_07_cluster
date: 2024-07-31
image: 
categories:
    - Redis
tags:
    - redis
    - docker
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---


### 搭建基于主从复制模式的集群
#### 主从复制模式概述
主服务器（Master Server）：负责处理客户端的写请求，写入数据后，会复制到一台或多台Redis服务器。  
从服务器（Slave Server）：负责处理客户端的读请求，从主服务器复制数据。  

主从复制模式的优点：  
1. 写操作集中在主服务器，读操作集中到从服务器，从而实现了读写分离，提升了读写性能。
2. 因为存在数据备份，因此能提升数据的安全性，当主服务器故障时，可以很快切换到从服务器读取数据。  

如果项目中并发要求不高，或者哪怕从Redis中读不到数据对性能也不会有太大损害，就可以使用一主一从。

**注意要点**：
1. 一个主服务器可以带一个或多个从服务器，从服务器可以再带从服务器，但在复制时只能把主服务器上的数据复制到从服务器。
2. 一台从服务器只能跟随一台主服务器，不能出现一从多主的模式。
3. Redis2.8以后的版本，采用异步的复制模式，进行主从复制时不会影响主服务器上的读写操作。

#### 用命令搭建基于主从复制模式的集群
这里搭建一个一主二从模式的集群。
```sh
# 创建主服务器master容器，占用6379端口
PS D:\code\blogs\farb.github.io> docker run -itd --name master -p 6379:6379 redis
ee645f3b1c2a0c56f5eafb5766701e04c638a4b5b26556cf27735ac4f3454618

# 创建从服务器slave1容器，占用6380端口，因为这是模拟一主多从模式，所以宿主机的端口号必须区分开
# 实际项目中，一般多台Redis服务器会部署在不同的服务器上，所以都可以使用6379端口
PS D:\code\blogs\farb.github.io> docker run -itd --name slave1 -p 6380:6380 redis
1d054285a7754e35c130b9951d8f3b2e473c8f989a422c993068cff47d1850f6

# 创建从服务器slave2容器，占用6381端口
PS D:\code\blogs\farb.github.io> docker run -itd --name slave2 -p 6381:6381 redis
c1b27efb7e6c112a83fac13fa1df4f8376faa291885933487d46478f260c568c

# 通过docker inspect master可以查看主服务器的详细信息，通过IPAddress可知主服务器容器的IP地址是172.17.0.2
# 真实项目中Redis服务器的IP一般是固定的，而通过Docker容器启动的Redis服务器的IP是动态的，可以通过docker inspect 查看

# 进入master容器，通过redis-cli命令进入Redis客户端，通过info replication命令查看Redis服务器的主从模式状态信息
PS D:\code\blogs\farb.github.io> docker exec -it master bash
root@ee645f3b1c2a:/data# redis-cli
127.0.0.1:6379> info replication
# Replication
# 可以看出角色是主服务器
role:master
# 可看出没有从服务器连接
connected_slaves:0
master_failover_state:no-failover
master_replid:821b97303c65400fb0d83fcc05b01545326b6083
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:0
second_repl_offset:-1
repl_backlog_active:0
repl_backlog_size:1048576
repl_backlog_first_byte_offset:0
repl_backlog_histlen:0

# 打开一个新的命令行窗口，进入slave1容器，通过redis-cli命令进入Redis客户端，通过info replication命令查看Redis服务器的主从模式状态信息
PS D:\code\blogs\farb.github.io> docker exec -it slave1 bash
root@1d054285a775:/data# redis-cli
127.0.0.1:6379> info replication
# Replication
# 可以看出和主服务器的信息一样，因为还没有设置主从模式，故此时还没区分主从服务器
role:master
connected_slaves:0
master_failover_state:no-failover
master_replid:59c0aa90a59be9ee610e0ea16f337f69e5116f1a
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:0
second_repl_offset:-1
repl_backlog_active:0
repl_backlog_size:1048576
repl_backlog_first_byte_offset:0
repl_backlog_histlen:0

# 在slave1容器中，通过slaveof命令设置slave1容器为master容器的从服务器，通过info replication命令查看Redis服务器的主从模式状态信息
127.0.0.1:6379> slaveof 172.17.0.2 6379
OK
127.0.0.1:6379> info replication
# Replication
# 可以看出角色是slave服务器，并且可以知道master服务器的IP地址和端口号
role:slave
master_host:172.17.0.2
master_port:6379
master_link_status:up
master_last_io_seconds_ago:8
#... 其他显示省略

# 再次进入master容器中，可以看到master容器中已经有一个从服务器了，并且能看到从服务器的详细信息
127.0.0.1:6379> info replication
# Replication
role:master
connected_slaves:1
slave0:ip=172.17.0.3,port=6379,state=online,offset=238,lag=1

# 最后进入slave2容器中，通过slaveof命令设置slave2容器为master容器的从服务器，通过info replication命令查看Redis服务器的主从模式状态信息
PS D:\code\blogs\farb.github.io> docker exec -it slave2 bash
root@c1b27efb7e6c:/data# redis-cli
127.0.0.1:6379> slaveof 172.17.0.2 6379
OK
127.0.0.1:6379> info replication
# Replication
# 可以看出角色是slave服务器，并且可以知道master服务器的IP地址和端口号
role:slave
master_host:172.17.0.2
master_port:6379

# 再次进入master容器中，可以看到master容器中已经有两个从服务器了，并且能看到从服务器的详细信息
127.0.0.1:6379> info replication
# Replication
role:master
connected_slaves:2
slave0:ip=172.17.0.3,port=6379,state=online,offset=630,lag=0
slave1:ip=172.17.0.4,port=6379,state=online,offset=630,lag=0

# 此时，在两个从服务器中执行get name,都返回nil，
# 当在主服务器中执行set name farb，两个从服务器中执行get name，都返回farb
```

#### 通过配置文件搭建主从集群
**除了可以使用slaveof命令搭建主从模式的集群，也可以使用redis.conf配置文件来搭建主从模式的集群。**

```sh
# 强制删除上面创建的一主而从的容器，防止影响后面的实验
PS D:\code\blogs\farb.github.io> docker rm -f  master slave1 slave2
master
slave1
slave2

# 创建主服务器master容器，占用6379端口,通过docker inpect命令查看主服务器的详细信息，通过IPAddress可知主服务器容器的IP地址是172.17.0.2
PS D:\code\blogs\farb.github.io> docker run -itd --name redis-master -p 6379:6379 redis

# 在redisSlave1.conf中配置如下：
port 6380 # redis-slave1服务器的端口号
slaveof 172.17.0.2 6379 # redis-slave1服务器的master服务器的IP地址和端口号

# 创建从服务器redis-slave1容器,并使用-v绑定redisSlave1.conf文件到容器中
PS D:\code\blogs\farb.github.io> docker run -itd --name redis-slave1 -v D:\ArchitectPracticer\Redis\RedisConfMasterSlave:/redisConfig:rw redis redis-server /redisConfig/redisSlave1.conf
1daa79fc05a9087ad4167ec5c4a5899935fbbb27c354667c94b0c8bf1100d207

# 进入redis-slave1容器中，通过redis-cli命令进入Redis客户端，通过info replication命令查看Redis服务器的主从模式状态信息
PS D:\code\blogs\farb.github.io> docker exec -it redis-slave1 bash

# redics-cli 默认使用的地址是127.0.0.1:6379，所以会连接失败
root@1daa79fc05a9:/data# redis-cli
Could not connect to Redis at 127.0.0.1:6379: Connection refused
not connected>
# 当前redisSlave1.conf中配置的端口是6380，所以需要通过-h指定主机地址，通过-p指定端口号
root@1daa79fc05a9:/data# redis-cli -h localhost -p 6380

# 可以看出角色是slave服务器，并且可以知道master服务器的IP地址和端口号
localhost:6380> info replication
# Replication
role:slave
master_host:172.17.0.2

# 同样的方式创建redis-slave2服务器，在redisSlave2.conf中配置如下：
port 6381
slaveof 172.17.0.2 6379

# 创建从服务器redis-slave2容器,并使用-v绑定redisSlave2.conf文件到容器中
PS D:\code\blogs\farb.github.io> docker run -itd --name redis-slave2 -v D:\ArchitectPracticer\Redis\RedisConfMasterSlave:/redisConfig:rw redis redis-server /redisConfig/redisSlave2.conf
8636a97f6e2d7ba10231b330d248f4e37ff2f2477ae74c87d1aae18227f44c99

# 进入redis-slave2容器中，通过redis-cli命令进入Redis客户端，通过info replication命令查看Redis服务器的主从模式状态信息
PS D:\code\blogs\farb.github.io> docker exec -it redis-slave2 bash
root@8636a97f6e2d:/data# redis-cli -h localhost -p 6381
localhost:6381> info replication
# Replication
role:slave
master_host:172.17.0.2
master_port:6379

# 再次回到redis-master容器中，通过info replication命令查看Redis服务器的主从模式状态信息
# 可以看到已经和redis-slave1和redis-slave2建立了主从模式的关系
127.0.0.1:6379> info replication
# Replication
role:master
connected_slaves:2
slave0:ip=172.17.0.3,port=6380,state=online,offset=14,lag=0
slave1:ip=172.17.0.4,port=6381,state=online,offset=14,lag=0

# 此时，在redis-master执行set age 18，redis-slave1和redis-slave2执行get age，都返回18
```