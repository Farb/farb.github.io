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
#### 配置读写分离效果
> 默认搭建好主从复制模式的集群后， 主服务器是可读可写的，而从服务器是只读的。如果对从服务器执行set等写命令，会报错。当然，如果确实需要，也可以将从服务器的只读属性设置为可读可写。

```sh   
# 进入redis-slave1容器中，通过redis-cli命令进入Redis客户端，通过info replication命令查看Redis服务器的主从模式状态信息
localhost:6380> info replication
# Replication
role:slave
master_host:172.17.0.2
master_port:6379
master_link_status:down
master_last_io_seconds_ago:-1
master_sync_in_progress:0
slave_read_repl_offset:14
slave_repl_offset:14
master_link_down_since_seconds:-1
slave_priority:100
# 可以看到slave_read_only属性为1，表示从服务器是只读的
slave_read_only:1
replica_announced:1
connected_slaves:0
master_failover_state:no-failover
master_replid:0d4ffddabfc229c371de8e1ee4d411424ad14261
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:14
second_repl_offset:-1
repl_backlog_active:0
repl_backlog_size:1048576
repl_backlog_first_byte_offset:0
repl_backlog_histlen:0

# 默认为只读模式，所以无法执行写操作
localhost:6380> set sex male
(error) READONLY You can't write against a read only replica.

# 通过redis.conf配置文件，将slave-read-only属性设置为no，表示从服务器是可读可写的
slave-read-only no

#重启redis-slave1容器，通过redis-cli命令进入Redis客户端，并再次执行set sex male
localhost:6380> get sex
(nil)
localhost:6380> set sex male
OK
localhost:6380> get sex
"male"

# 分别进入redis-master和redis-slave2容器中，执行get sex均返回nil,说明写入从服务器的数据不会进行任何同步操作。
```

#### 用心跳机制提高主从复制的可靠性
> 主从复制的模式里，从服务器会默认以一秒一次的频率向主服务器发送REPLCONF ACK命令，来确保主服务器和从服务器之间的连接。这种定时交互命令确保连接的机制叫做“心跳”机制。在主服务器中执行info replication命令，可以看到从属于它的从服务器的“心跳”状态

```sh
127.0.0.1:6379> info replication
# Replication
role:master
connected_slaves:2
slave0:ip=172.17.0.3,port=6380,state=online,offset=14,lag=0
slave1:ip=172.17.0.4,port=6381,state=online,offset=14,lag=0

# 通过上面的lag=0，可知主从服务器之间连接畅通，延迟都是0秒

# 创建主服务器的配置文件redisMaster.conf，配置如下：
min-slaves-to-write 2
# 连接到主服务器的从服务器最少需要2个，否则主服务器将拒绝同步操作

min-slaves-max-lag 15
# 从服务器与主服务器之间的最大延迟不能超过15秒

# 以上两个条件是“或”的关系，只要一个条件不满足就停止主从复制的操作。

# 重新创建并启动redis-master容器，并使用-v绑定redisMaster.conf文件到容器中
PS D:\code\blogs\farb.github.io> docker run -itd --name redis-master -v D:\ArchitectPracticer\Redis\RedisConfMasterSlave:/redisConfig:rw -p 6379:6379 redis redis-server /redisConfig/redisMaster.conf 
24ea09c6e60202c13e04ff52339e397dcc539da6376444e03d9fe66e58be59d3

# 主服务器写入set k1 v1 成功后，redis-slave1和redis-slave2均可读取数据
127.0.0.1:6379> set k1 v1
OK

# shutdown redis-slave2服务器后，再次在主服务器执行set命令，发现报错。因为违反了配置中的min-slaves-to-write 2
127.0.0.1:6379> set k2 v2
(error) NOREPLICAS Not enough good replicas to write.
```

#### 用偏移量检查数据是否一致
**master_repl_offset**：主服务器向从服务器发送的字节数  
**slave_repl_offset**: 从服务器接收到来来自主服务器发送的字节数 

```sh
# 进入redis-master容器中，通过redis-cli命令进入Redis客户端，通过info replication命令查看Redis服务器的主从模式状态信息
# 可以看到主服务器向从服务器发送了字节数906
127.0.0.1:6379> info replication
# Replication
role:master
connected_slaves:2
min_slaves_good_slaves:2
slave0:ip=172.17.0.2,port=6380,state=online,offset=906,lag=1
slave1:ip=172.17.0.4,port=6381,state=online,offset=906,lag=1
...
master_repl_offset:906

# 分别进入redis-slave1容器中，通过redis-cli命令进入Redis客户端，通过info replication命令查看Redis服务器的主从模式状态信息
# 应该可以看到slave_repl_offset的值也是906（前提是没有对从服务器执行写操作，且主从服务器数据一直同步，否则可能会不一致）
```
### 搭建哨兵模式的集群
> 基于主从复制+读写分离模式的集群，一旦主服务器发生故障，则需要手动切换到从服务器上，同时需要重新设置主从关系。如果采用哨兵模式，则不需要手动切换，当主服务器发生故障时，哨兵会自动切换到从服务器上，实现“故障自动恢复”的效果，保证集群的高可用性。

#### 哨兵模式概述
> 哨兵机制一般和主从复制模式整合使用，在基于哨兵的模式里会在一台或多台服务器上引入哨兵进程，这些节点叫哨兵节点。
> 哨兵节点一般不存储数据，主要作用是监控主从模式里的主服务器节点(哨兵节点之间也相互监控)。当哨兵节点监控的主服务器发生故障时，哨兵节点会主导“故障自动恢复”的过程，具体来讲就是会通过选举模式在该主服务器下属的从服务器中选择一个作为新的主服务器，并完成相应的数据和配置等更改操作。
> 基于哨兵模式的集群，可以让故障自动恢复，从而提升系统的可用性。实际项目中一般会配置多个主从模式集群，所以需要引入多个哨兵节点。

### 搭建哨兵模式集群实战
```sh
# 清理环境，删除所有容器避免影响，使用 docker rm $(docker ps -aq) 或 docker container prune -f 命令
PS D:\code\blogs\farb.github.io> docker rm $(docker ps -aq)
2f25343c996e
aa62f3456cb4
25bd36e835b9

# 按照上面的实战，再次创建一个一主二从的主从复制模式
PS D:\code\blogs\farb.github.io> docker run -itd --name redis-master -v D:\ArchitectPracticer\Redis\RedisConfSentinel:/redisConfig:rw -p 6379:6379 redis redis-server /redisConfig/redisMaster.conf
8227be97fad3ca13872c2c304d0f49fd918521ed31ca5ea51914d9bd7ed701a6

PS D:\code\blogs\farb.github.io> docker run -itd --name redis-slave1 -v D:\ArchitectPracticer\Redis\RedisConfSentinel:/redisConfig:rw -p 6380:6380 redis redis-server /redisConfig/redisSlave1.conf
4ce5f884fb26bd3760047e4c93b95356b3b70048f423c2fdff40e8cb8ec76b37
PS D:\code\blogs\farb.github.io> docker run -itd --name redis-slave2 -v D:\ArchitectPracticer\Redis\RedisConfSentinel:/redisConfig:rw -p 6381:6381 redis redis-server /redisConfig/redisSlave2.conf
6470580ea30f9903aa64b7d54ca5ecbd8777cbef0cea8ecd891cdcd07337793a

# 进入redis-master容器中，通过redis-cli命令进入Redis客户端，通过info replication命令查看Redis服务器的主从模式状态信息,确保主从复制模式已经搭建成功
PS D:\code\blogs\farb.github.io> docker exec -it redis-master bash
root@8227be97fad3:/data# redis-cli
127.0.0.1:6379> info replication
# Replication
role:master
connected_slaves:2
min_slaves_good_slaves:2
slave0:ip=172.17.0.3,port=6380,state=online,offset=196,lag=1
slave1:ip=172.17.0.4,port=6381,state=online,offset=196,lag=0
master_failover_state:no-failover
master_replid:65bf7b2a0d40228c859adf93672af1fa88601443
master_replid2:0000000000000000000000000000000000000000

```

**基础的主从复制模式搭建完成后，下面开始哨兵的搭建**

```sh
port 16379
# 哨兵节点的工作端口是16379

sentinel monitor master 172.17.0.2 6379 2
# 哨兵节点监控主服务器，主机ip是172.17.0.2，端口是6379，集群中至少2个哨兵节点才能确定该主服务器是否故障

dir /
logfile "sentinel1.log"
# 哨兵节点日志文件位置和名称

# 创建并启动哨兵节点容器，通过-v挂载了D:\ArchitectPracticer\Redis\RedisConfSentinel目录
PS D:\code\blogs\farb.github.io> docker run -itd --name sentinel1 -v D:\ArchitectPracticer\Redis\RedisConfSentinel:/redisConfig:rw -p 16379:16379 redis redis-sentinel /redisConfig/sentinel1.conf   
59caeda82a6111e9e1c324e0c580e2591f4a4c7ba3dcc52d4f5e3cddb9c6e463

# 进入sentinel1容器中，通过redis-cli命令进入Redis客户端，通过info sentinel命令查看Redis服务器的哨兵模式状态信息
PS D:\code\blogs\farb.github.io> docker exec -it sentinel1 bash
root@59caeda82a61:/data# redis-cli -h localhost -p 16379
localhost:16379> info sentinel
# Sentinel
sentinel_masters:1
sentinel_tilt:0
sentinel_tilt_since_seconds:-1
sentinel_running_scripts:0
sentinel_scripts_queue_length:0
sentinel_simulate_failure_flags:0
master0:name=master,status=ok,address=172.17.0.2:6379,slaves=2,sentinels=1
# 最后一句可以看到该主服务器的状态是ok，说明哨兵节点已经成功监控到主服务器，并且该主服务器的ip和端口也可看到，该主服务器下有2个从服务器，并且有一个哨兵节点


# 同样地，创建sentinel2.conf配置文件
port 16380
sentinel resolve-hostnames yes
sentinel monitor master 172.17.0.2 6379 2
dir "/"
logfile "sentinel2.log"

# 创建并启动第二个哨兵节点容器，通过-v挂载了D:\ArchitectPracticer\Redis\RedisConfSentinel目录
docker run -itd --name sentinel2 -v D:\ArchitectPracticer\Redis\RedisConfSentinel:/redisConfig:rw -p 16380:16380 redis redis-sentinel /redisConfig/sentinel2.conf      
2a742a70e80989b868a394856e95d97c58be40e1b892025463ab22646f85c759
PS D:\code\blogs\farb.github.io> docker exec -it sentinel2 bash
root@2a742a70e809:/data# redis-cli -h localhost -p 16380
localhost:16380> info sentinel
# Sentinel
sentinel_masters:1
sentinel_tilt:0
sentinel_tilt_since_seconds:-1
sentinel_running_scripts:0
sentinel_scripts_queue_length:0
sentinel_simulate_failure_flags:0
master0:name=master,status=ok,address=172.17.0.2:6379,slaves=2,sentinels=2
# 最后一句可以看到该主服务器的状态是ok，说明哨兵节点已经成功监控到主服务器，并且该主服务器的ip和端口也可看到，该主服务器下有2个从服务器，并且有2个哨兵节点

```

**至此，基于哨兵的一主二从的Redis集群已经搭建完成：两个哨兵节点sentinel1和sentinel2同时监控redis-master主服务器，主服务器下挂着两个从服务器redis-slave1和redis-slave2，并且主服务器和两个从服务器之间依然存在“主从复制模式”。实际项目里可以让哨兵节点监控多个集群**

#### 哨兵节点的常用配置
```sh
# 哨兵节点监控主服务器后，如果主服务器挂了，需要多久才能确定该主服务器已经故障 ，
# master是哨兵节点监控的服务器名称，需要和sentinel monitor命令中设置的服务器名称master一致
# 之前配置了 sentinel monitor master 172.17.0.2 6379 2，因而至少2个哨兵节点都超过60s没有收到master的响应，才能确定该主服务器已经故障
sentinel down-after-milliseconds master 60000

# 哨兵节点监控主服务器后，如果主服务器故障了，必须在180s内自动切换到其他从服务器，否则认定本次恢复动作失败。
sentinel failover-timeout master 180000
```

#### 哨兵模式下的故障自动恢复效果
**将redis-server容器停止或通过shutdown命令关闭redis-master，通过info sentinel命令观察哨兵节点的日志，通过info replication命令观察主从复制的状态信息，来确认哨兵节点的故障自动恢复效果**
```sh
# 关闭redis-master容器
127.0.0.1:6379> shutdown

# 进入sentinel1容器，通过info sentinel命令观察哨兵节点的日志，发现主服务器已经由之前的主服务器172.17.0.2:6379，变成了新的主服务器172.17.0.3:6380
localhost:16379> info sentinel
# Sentinel
sentinel_masters:1
sentinel_tilt:0
sentinel_tilt_since_seconds:-1
sentinel_running_scripts:0
sentinel_scripts_queue_length:0
sentinel_simulate_failure_flags:0
master0:name=master,status=ok,address=172.17.0.3:6380,slaves=2,sentinels=2

#进入sentinel2容器，能够看到相同的信息。

# 进入redis-slave1容器，通过info replication命令观察主从复制的状态信息，发现redis-slave1已经成为主服务，并且之前的redis-slave2已经成为从服务器
localhost:6380> info replication
# Replication
role:master
connected_slaves:1
slave0:ip=172.17.0.4,port=6381,state=online,offset=315538,lag=0

# 进入redis-slave2容器，通过info replication命令观察主从复制的状态信息，也发现redis-slave1已经成为主服务器
redis-cli -h localhost -p 6381
localhost:6381> info replication
# Replication
role:slave
master_host:172.17.0.3
master_port:6380
master_link_status:up
```
**通过上述实践，发现故障已经自动恢复**

#### 通过日志观察故障恢复流程
通过观察哨兵的日志，查看故障自动恢复的流程

关键词：
- sdown：主观宕机subjective down
- odown: 客观宕机objective down
- failover: 故障转移
- vote-for-leader： 领导选举

```sh
# 先查看sentinel2.log日志
root@2a742a70e809:/data# cat /sentinel2.log
1:X 03 Aug 2024 03:44:13.749 * oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
1:X 03 Aug 2024 03:44:13.749 * Redis version=7.2.5, bits=64, commit=00000000, modified=0, pid=1, just started
1:X 03 Aug 2024 03:44:13.749 * Configuration loaded
1:X 03 Aug 2024 03:44:13.749 * monotonic clock: POSIX clock_gettime
1:X 03 Aug 2024 03:44:13.749 * Running mode=sentinel, port=16380.
1:X 03 Aug 2024 03:44:13.749 * Sentinel ID is 750555c54a6c4e294749888502b1644dfd0f3ac7
1:X 03 Aug 2024 03:44:13.749 # +monitor master master 172.17.0.2 6379 quorum 2
1:X 03 Aug 2024 03:44:13.751 * +slave slave 172.17.0.3:6380 172.17.0.3 6380 @ master 172.17.0.2 6379
1:X 03 Aug 2024 03:44:13.756 * Sentinel new configuration saved on disk
1:X 03 Aug 2024 03:44:13.756 * +slave slave 172.17.0.4:6381 172.17.0.4 6381 @ master 172.17.0.2 6379
1:X 03 Aug 2024 03:44:13.760 * Sentinel new configuration saved on disk
1:X 03 Aug 2024 03:44:14.780 * +sentinel sentinel fc41e2130b8b8896db18204e1b1ca868161369e2 172.17.0.5 16379 @ master 172.17.0.2 6379
1:X 03 Aug 2024 03:44:14.784 * Sentinel new configuration saved on disk

# 主服务器redis-master被sentinel2节点主观下线
1:X 03 Aug 2024 04:18:54.656 # +sdown master master 172.17.0.2 6379
# 主服务器redis-master被sentinel2节点客观下线，因为此时sentinel1节点也认为主观下线，共有两个节点满足主观下线条件quorum 2/2，从而客观下线
1:X 03 Aug 2024 04:18:54.727 # +odown master master 172.17.0.2 6379 #quorum 2/2
1:X 03 Aug 2024 04:18:54.727 # +new-epoch 1
# 尝试故障转移
1:X 03 Aug 2024 04:18:54.727 # +try-failover master master 172.17.0.2 6379
1:X 03 Aug 2024 04:18:54.732 * Sentinel new configuration saved on disk
# 领导选举
1:X 03 Aug 2024 04:18:54.732 # +vote-for-leader 750555c54a6c4e294749888502b1644dfd0f3ac7 1
1:X 03 Aug 2024 04:18:54.738 * fc41e2130b8b8896db18204e1b1ca868161369e2 voted for 750555c54a6c4e294749888502b1644dfd0f3ac7 1
1:X 03 Aug 2024 04:18:54.815 # +elected-leader master master 172.17.0.2 6379
1:X 03 Aug 2024 04:18:54.815 # +failover-state-select-slave master master 172.17.0.2 6379
1:X 03 Aug 2024 04:18:54.887 # +selected-slave slave 172.17.0.3:6380 172.17.0.3 6380 @ master 172.17.0.2 6379
1:X 03 Aug 2024 04:18:54.887 * +failover-state-send-slaveof-noone slave 172.17.0.3:6380 172.17.0.3 6380 @ master 172.17.0.2 6379
1:X 03 Aug 2024 04:18:54.958 * +failover-state-wait-promotion slave 172.17.0.3:6380 172.17.0.3 6380 @ master 172.17.0.2 6379
1:X 03 Aug 2024 04:18:55.604 * Sentinel new configuration saved on disk
1:X 03 Aug 2024 04:18:55.604 # +promoted-slave slave 172.17.0.3:6380 172.17.0.3 6380 @ master 172.17.0.2 6379
1:X 03 Aug 2024 04:18:55.604 # +failover-state-reconf-slaves master master 172.17.0.2 6379
1:X 03 Aug 2024 04:18:55.662 * +slave-reconf-sent slave 172.17.0.4:6381 172.17.0.4 6381 @ master 172.17.0.2 6379
1:X 03 Aug 2024 04:18:55.880 # -odown master master 172.17.0.2 6379
1:X 03 Aug 2024 04:18:56.656 * +slave-reconf-inprog slave 172.17.0.4:6381 172.17.0.4 6381 @ master 172.17.0.2 6379
1:X 03 Aug 2024 04:18:56.656 * +slave-reconf-done slave 172.17.0.4:6381 172.17.0.4 6381 @ master 172.17.0.2 6379
1:X 03 Aug 2024 04:18:56.739 # +failover-end master master 172.17.0.2 6379
# 切换主服务器到新的主服务器172.17.0.3:6380
1:X 03 Aug 2024 04:18:56.739 # +switch-master master 172.17.0.2 6379 172.17.0.3 6380
1:X 03 Aug 2024 04:18:56.739 * +slave slave 172.17.0.4:6381 172.17.0.4 6381 @ master 172.17.0.3 6380
1:X 03 Aug 2024 04:18:56.739 * +slave slave 172.17.0.2:6379 172.17.0.2 6379 @ master 172.17.0.3 6380
1:X 03 Aug 2024 04:18:56.743 * Sentinel new configuration saved on disk
1:X 03 Aug 2024 04:19:26.747 # +sdown slave 172.17.0.2:6379 172.17.0.2 6379 @ master 172.17.0.3 6380



# 查看senttinel1.log日志
# 主服务器redis-master被sentinel1节点主观下线,可以看到比sentinel2节点快了1毫秒
1:X 03 Aug 2024 04:18:54.655 # +sdown master master 172.17.0.2 6379
1:X 03 Aug 2024 04:18:54.735 * Sentinel new configuration saved on disk
1:X 03 Aug 2024 04:18:54.735 # +new-epoch 1
1:X 03 Aug 2024 04:18:54.738 * Sentinel new configuration saved on disk
# 领导选举
1:X 03 Aug 2024 04:18:54.738 # +vote-for-leader 750555c54a6c4e294749888502b1644dfd0f3ac7 1
1:X 03 Aug 2024 04:18:55.663 # +config-update-from sentinel 750555c54a6c4e294749888502b1644dfd0f3ac7 172.17.0.6 16380 @ master 172.17.0.2 6379
# 切换主服务器到新的主服务器172.17.0.3:6380
1:X 03 Aug 2024 04:18:55.663 # +switch-master master 172.17.0.2 6379 172.17.0.3 6380
1:X 03 Aug 2024 04:18:55.663 * +slave slave 172.17.0.4:6381 172.17.0.4 6381 @ master 172.17.0.3 6380
1:X 03 Aug 2024 04:18:55.663 * +slave slave 172.17.0.2:6379 172.17.0.2 6379 @ master 172.17.0.3 6380

# 由于只能由一个哨兵节点完成故障自动恢复，因此如果有多个哨兵节点同时监控到主服务器失效，那么最终只有一个哨兵节点通过竞争得到故障恢复的权力。
# 从上面的日志中，可以看到故障恢复的权力已经被sentinel2节点竞争得到，sentinel1只能在sdown之后处于停留状态，待sentinel2节点完成故障恢复之后重新切换到新的主服务器节点，继续监控新的主服务器节点和对应的从服务器节点  。
```
#### 故障节点恢复后的表现
从上面的日志中可以看到redis-master服务器处于失效状态，但是在新的主从复制集群里会把该服务器当作从服务器。再重新启动redis-master容器，模拟故障排除后的效果。

```sh
# 可以看到，redis-master重启后，自动以slave的身份接入。
redis-cli -h localhost -p 6380
localhost:6380> info replication
# Replication
role:master
connected_slaves:2
slave0:ip=172.17.0.4,port=6381,state=online,offset=3096199,lag=1
slave1:ip=172.17.0.2,port=6379,state=online,offset=3096199,lag=1
```

**由上可知，哨兵节点不仅能自动恢复故障，而且当故障节点恢复后，会自动把它加入到集群中，而无需人工干预。与简单的主从复制模式集群相比，哨兵模式的集群能更好地提升系统的可靠性。**

### 搭建Cluster集群
> 相比哨兵模式，cluster集群能支持扩容，且无需额外的节点来监控状态。

#### 哈希槽和cluster集群
**cluster集群里有16384个哈希槽（hash slot），在写入数据时，会先用CRC16算法对key进行运算，并用16384对运算结果进行模运算，最终结果作为哈希槽的索引，将数据存入对应的哈希槽中。`slotIndex=Hash_Slot=CRC16(key) % 16384`。这里说的cluster集群有16384个哈希槽，并不意味着集群中一定要有16374个节点，哈希槽是虚拟的，是会被分配到若干台集群中的机器中的。**

比如，某cluster集群由三台Redis服务器组成，平均每台服务器可以分配16384/3=5461个哈希槽。由于哈希槽编号从0开始，所以编号[0,5460]会分配给第一台服务器，编号[5461,10922]分配给第二台服务器，[10923,16383]分配给第三台服务器。同理，如果有6台Redis服务器，平均每个服务器可以分配16384/6=2730个哈希槽。此外，cluster也支持主从复制，即分配到一定数量的Redis主服务器也可以携带一个或多个从服务器。如下图是一个三主三从的cluster集群。

![](https://s3.bmp.ovh/imgs/2024/08/04/557c7a6fd895766b.png)

#### 初步搭建cluster集群
搭建如上图所示的三主三从的cluster集群，其他类型的依次类推。

** 1.首先创建三主三从服务器的配置文件**

```sh
# 主服务器1的配置文件 clusterMaster1.conf
port 6379
dir /redisConfig
# 日志存放目录
logfile clusterMaster1.log
# 日志文件名
cluster-enabled yes
# 开启集群模式
cluster-config-file nodes-clusterMaster1.conf
# 集群相关的配置文件，会自动生成

# 主服务器2的配置文件 clusterMaster2.conf
port 6380
dir /redisConfig
logfile clusterMaster2.log
cluster-enabled yes
cluster-config-file nodes-clusterMaster2.conf

# 主服务器3的配置文件 clusterMaster3.conf
port 6381
dir /redisConfig
logfile clusterMaster3.log
cluster-enabled yes
cluster-config-file nodes-clusterMaster3.conf

# 从服务器1的配置文件 clusterSlave1.conf
port 16379
dir /redisConfig
logfile clusterSlave1.log
cluster-enabled yes
cluster-config-file nodes-clusterSlaver1.conf

# 从服务器2的配置文件 clusterSlave2.conf
port 16380
dir /redisConfig
logfile clusterSlave2.log
cluster-enabled yes
cluster-config-file nodes-clusterSlave2.conf

# 从服务器3的配置文件 clusterSlave3.conf
port 16381
dir /redisConfig
logfile clusteSlave3.log
cluster-enabled yes
cluster-config-file nodes-clusterSlave3.conf
```

**2. 启动集群中的每个容器**
```sh
# 分别启动clusterMaster1 clusterMaster2 clusterMaster3 clusterSlave1 clusterSlave2 clusterSlave3 6个容器
# 每个容器只是容器名称，端口号和配置文件不同
PS D:\code\blogs\farb.github.io> docker run -itd --name clusterMaster1 -v D:\ArchitectPracticer\Redis\RedisConfCluster:/redisConfig:rw -p 6379:6379 redis redis-server /redisConfig/clusterMaster1.conf  
fb368ad5ad39950afced6ca59c9be133d8440623f0f4aedd008cec2b6bcdd735

PS D:\code\blogs\farb.github.io> docker run -itd --name clusterMaster2 -v D:\ArchitectPracticer\Redis\RedisConfCluster:/redisConfig:rw -p 6380:6380 redis redis-server /redisConfig/clusterMaster2.conf
ccb9bf970d30c6273b5d7b9e82d3729520c53cb440a49766cc4e0c21f55d23d8

PS D:\code\blogs\farb.github.io> docker run -itd --name clusterMaster3 -v D:\ArchitectPracticer\Redis\RedisConfCluster:/redisConfig:rw -p 6381:6381 redis redis-server /redisConfig/clusterMaster3.conf
dcd5903265bf75182ded211dfb7728f97fc734f926bcba813332fe7cdb898aeb

PS D:\code\blogs\farb.github.io> docker run -itd --name clusterSlave1 -v D:\ArchitectPracticer\Redis\RedisConfCluster:/redisConfig:rw -p 16379:16379 redis redis-server /redisConfig/clusterSlave1.conf 
5d2ecf9131f7069cf410940fad6a85a48e7292fac4b709cc60116ebd57ffca92

PS D:\code\blogs\farb.github.io> docker run -itd --name clusterSlave2 -v D:\ArchitectPracticer\Redis\RedisConfCluster:/redisConfig:rw -p 16380:16380 redis redis-server /redisConfig/clusterSlave2.conf
44e2e24b798af85be78c2c364e15440cf11447a645567d527833a5b95548dbef

PS D:\code\blogs\farb.github.io> docker run -itd --name clusterSlave3 -v D:\ArchitectPracticer\Redis\RedisConfCluster:/redisConfig:rw -p 16381:16381 redis redis-server /redisConfig/clusterSlave3.conf
ef29a379b55728b20833f4d2b90846d8de034ae9652a76c0acedbd80c53ee5ce

# 查看容器列表，确保6个容器都处于Up状态
PS D:\code\blogs\farb.github.io> docker ps 
CONTAINER ID   IMAGE     COMMAND                   CREATED         STATUS         PORTS                                NAMES
ef29a379b557   redis     "docker-entrypoint.s…"   3 minutes ago   Up 3 minutes   6379/tcp, 0.0.0.0:16381->16381/tcp   clusterSlave3
44e2e24b798a   redis     "docker-entrypoint.s…"   4 minutes ago   Up 4 minutes   6379/tcp, 0.0.0.0:16380->16380/tcp   clusterSlave2
5d2ecf9131f7   redis     "docker-entrypoint.s…"   4 minutes ago   Up 4 minutes   6379/tcp, 0.0.0.0:16379->16379/tcp   clusterSlave1
dcd5903265bf   redis     "docker-entrypoint.s…"   5 minutes ago   Up 5 minutes   6379/tcp, 0.0.0.0:6381->6381/tcp     clusterMaster3
ccb9bf970d30   redis     "docker-entrypoint.s…"   6 minutes ago   Up 6 minutes   6379/tcp, 0.0.0.0:6380->6380/tcp     clusterMaster2
fb368ad5ad39   redis     "docker-entrypoint.s…"   2 hours ago     Up 2 hours     0.0.0.0:6379->6379/tcp               clusterMaster1

# 打开nodes-clusterMaster1.conf文件，如下：
8b9e6e09898be857fcd33e4a2611c42268ce7b1c :0@0,,tls-port=0,shard-id=358d229be66be4999268253234d3bb58623dbe83 myself,master - 0 0 0 connected
vars currentEpoch 0 lastVoteEpoch 0

# 可以看到，当前节点属于master节点，只连接到myself自身，没同其他节点关联。其他nodes-*.conf文件类似，都是master节点，没有关联其他节点。稍后将用meet命令关联。
```

**3.使用redis-cli --cluster create命令创建集群**
**使用 `docker inspect -f '{{.NetworkSettings.IPAddress}}' containerName` 查看容器的IP地址**

```sh
PS D:\code\blogs\farb.github.io> docker inspect -f '{{.NetworkSettings.IPAddress}}' clusterMaster1
172.17.0.2
PS D:\code\blogs\farb.github.io> docker inspect -f '{{.NetworkSettings.IPAddress}}' clusterMaster2
172.17.0.3
PS D:\code\blogs\farb.github.io> docker inspect -f '{{.NetworkSettings.IPAddress}}' clusterMaster3
172.17.0.4
PS D:\code\blogs\farb.github.io> docker inspect -f '{{.NetworkSettings.IPAddress}}' clusterSlave1 
172.17.0.5
PS D:\code\blogs\farb.github.io> docker inspect -f '{{.NetworkSettings.IPAddress}}' clusterSlave2
172.17.0.6
PS D:\code\blogs\farb.github.io> docker inspect -f '{{.NetworkSettings.IPAddress}}' clusterSlave3
172.17.0.7
```
为便于查看，列出统计表格如下：
| 节点名称 | IP地址 |端口|查看Ip地址所用的Docker命令|
| --- | --- | --- | --- |
| clusterMaster1 | 172.17.0.2 | 6379 | docker inspect -f '{{.NetworkSettings.IPAddress}}' clusterMaster1 |
| clusterMaster2 | 172.17.0.3 | 6380 | docker inspect -f '{{.NetworkSettings.IPAddress}}' clusterMaster2 |
| clusterMaster3 | 172.17.0.4 | 6381 | docker inspect -f '{{.NetworkSettings.IPAddress}}' clusterMaster3 |
| clusterSlave1 | 172.17.0.5 | 16379 | docker inspect -f '{{.NetworkSettings.IPAddress}}' clusterSlave1 |
| clusterSlave2 | 172.17.0.6 | 16380 | docker inspect -f '{{.NetworkSettings.IPAddress}}' clusterSlave2 |
| clusterSlave3 | 172.17.0.7 | 16381 | docker inspect -f '{{.NetworkSettings.IPAddress}}' clusterSlave3 |

**之后，进入clusterMaster1节点，使用--cluster create命令连接各个节点，这样所有的节点就在一个集群了。这里我也使用过cluster meet命令，但是使用这种方式后扩容时，添加的节点一直是从节点，具体原因未知**

```sh
# 进入clusterMaster1节点，创建集群，注意此处还没有设置主从关系，这里设置的是--cluster-replicas 0
# 为的是想要创建6379(主)-16379(从)这样的对应关系，否则如果交给redis-cli自动处理的话，会出现端口端口随机配对的情况
PS D:\code\blogs\farb.github.io> docker exec -it clusterMaster1 bash
root@fb368ad5ad39:/data# redis-cli -p 6379 --cluster create --cluster-replicas 0 172.17.0.2:6379 172.17.0.3:6380 172.17.0.4:6381
>>> Performing hash slots allocation on 3 nodes...
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
M: b6d4a069fdd4acf4d46133e56bfa79cb6d03b77c 172.17.0.2:6379
   slots:[0-5460] (5461 slots) master
M: 8ba84e5b7c76e8530529b41d8c6ee59f3d3edf0c 172.17.0.3:6380
   slots:[5461-10922] (5462 slots) master
M: e2737cdc7073c3750ffb670bc618dc28cc939877 172.17.0.4:6381
   slots:[10923-16383] (5461 slots) master
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join
.
>>> Performing Cluster Check (using node 172.17.0.2:6379)
M: b6d4a069fdd4acf4d46133e56bfa79cb6d03b77c 172.17.0.2:6379
   slots:[0-5460] (5461 slots) master
M: 8ba84e5b7c76e8530529b41d8c6ee59f3d3edf0c 172.17.0.3:6380
   slots:[5461-10922] (5462 slots) master
M: e2737cdc7073c3750ffb670bc618dc28cc939877 172.17.0.4:6381
   slots:[10923-16383] (5461 slots) master
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.

# 从上面可以看出redis-cli会根据你输入的yes，自动给三个主节点平均分配哈希槽

# 然后查看clusterMaster节点的cluster info。
root@fb368ad5ad39:/data# redis-cli 
127.0.0.1:6379> cluster info
# 集群状态是fail
cluster_state:fail
# 分配的哈希槽为0
cluster_slots_assigned:0
cluster_slots_ok:0
cluster_slots_pfail:0
cluster_slots_fail:0
# 可以看到集群中有6个节点，但是集群状态是fail，说明集群还没有完成初始化。
# 原因是还没给每个节点分配哈希槽（Hash Slot）
cluster_known_nodes:6

# cluster nodes查看节点信息，可以看到三个主节点
127.0.0.1:6379> cluster nodes
8ba84e5b7c76e8530529b41d8c6ee59f3d3edf0c 172.17.0.3:6380@16380 master - 0 1723297213811 2 connected 5461-10922
e2737cdc7073c3750ffb670bc618dc28cc939877 172.17.0.4:6381@16381 master - 0 1723297212809 3 connected 10923-16383
b6d4a069fdd4acf4d46133e56bfa79cb6d03b77c 172.17.0.2:6379@16379 myself,master - 0 1723297211000 1 connected 0-5460
```

**4.为三个主节点分配从节点**    
```sh
# 为主节点172.17.0.2:6379配置从节点172.17.0.5:16379
# 172.17.0.5:16379为要添加的从节点IP和端口
# 172.17.0.2:6379为要添加的从节点所属集群的任意一个节点的IP和端口
# --cluster-master-id b6d4a069fdd4acf4d46133e56bfa79cb6d03b77c为要添加的从节点对应的主节点ID
root@fb368ad5ad39:/data# redis-cli -p 6379 --cluster add-node 172.17.0.5:16379 172.17.0.2:6379 --cluster-slave --cluster-master-id b6d4a069fdd4acf4d46133e56bfa79cb6d03b77c
>>> Adding node 172.17.0.5:16379 to cluster 172.17.0.2:6379
>>> Performing Cluster Check (using node 172.17.0.2:6379)
M: b6d4a069fdd4acf4d46133e56bfa79cb6d03b77c 172.17.0.2:6379
   slots:[0-5460] (5461 slots) master
M: 8ba84e5b7c76e8530529b41d8c6ee59f3d3edf0c 172.17.0.3:6380
   slots:[5461-10922] (5462 slots) master
M: e2737cdc7073c3750ffb670bc618dc28cc939877 172.17.0.4:6381
   slots:[10923-16383] (5461 slots) master
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
>>> Send CLUSTER MEET to node 172.17.0.5:16379 to make it join the cluster.
Waiting for the cluster to join
.
>>> Configure node as replica of 172.17.0.2:6379.
[OK] New node added correctly.

# 通过输出信息，可以看到添加从节点成功，也可以通过cluster nodes验证
127.0.0.1:6379> cluster nodes
8ba84e5b7c76e8530529b41d8c6ee59f3d3edf0c 172.17.0.3:6380@16380 master - 0 1723297831286 2 connected 5461-10922
e2737cdc7073c3750ffb670bc618dc28cc939877 172.17.0.4:6381@16381 master - 0 1723297830283 3 connected 10923-16383
b6d4a069fdd4acf4d46133e56bfa79cb6d03b77c 172.17.0.2:6379@16379 myself,master - 0 1723297828000 1 connected 0-5460
30e652ae7137b28a304a6954acc7dd99a070be08 172.17.0.5:16379@26379 slave b6d4a069fdd4acf4d46133e56bfa79cb6d03b77c 0 1723297832289 1 connected

# 按照如上步骤，依次添加另外两个从节点172.17.0.6:16380和172.17.0.7:16381
root@fb368ad5ad39:/data# redis-cli -p 6379 --cluster add-node 172.17.0.6:16380 172.17.0.2:6379 --cluster-slave --cluster-master-id 8ba84e5b7c76e8530529b41d8c6ee59f3d3edf0c 
root@fb368ad5ad39:/data# redis-cli -p 6379 --cluster add-node 172.17.0.7:16381 172.17.0.2:6379 --cluster-slave --cluster-master-id e2737cdc7073c3750ffb670bc618dc28cc939877 

# 执行成功后，可以看到三主三从的集群搭建成功
127.0.0.1:6379> cluster nodes
e2737cdc7073c3750ffb670bc618dc28cc939877 172.17.0.4:6381@16381 master - 0 1723298034325 3 connected 10923-16383
30e652ae7137b28a304a6954acc7dd99a070be08 172.17.0.5:16379@26379 slave b6d4a069fdd4acf4d46133e56bfa79cb6d03b77c 0 1723298033323 1 connected
808b97759fd358611f94edca21836c4d68d63467 172.17.0.6:16380@26380 slave 8ba84e5b7c76e8530529b41d8c6ee59f3d3edf0c 0 1723298035332 2 connected
fc156444e5af0817468c95f8a86ec55310e06aed 172.17.0.7:16381@26381 slave e2737cdc7073c3750ffb670bc618dc28cc939877 0 1723298033000 3 connected
b6d4a069fdd4acf4d46133e56bfa79cb6d03b77c 172.17.0.2:6379@16379 myself,master - 0 1723298033000 1 connected 0-5460
8ba84e5b7c76e8530529b41d8c6ee59f3d3edf0c 172.17.0.3:6380@16380 master - 0 1723298035000 2 connected 5461-10922

# 也可以这样查看 redis-cli --cluster check nodeHost:nodePort
root@fb368ad5ad39:/data# redis-cli --cluster check 172.17.0.2:6379
172.17.0.2:6379 (b6d4a069...) -> 0 keys | 5461 slots | 1 slaves.
172.17.0.4:6381 (e2737cdc...) -> 0 keys | 5461 slots | 1 slaves.
172.17.0.3:6380 (8ba84e5b...) -> 0 keys | 5462 slots | 1 slaves.
[OK] 0 keys in 3 masters.
0.00 keys per slot on average.
>>> Performing Cluster Check (using node 172.17.0.2:6379)
M: b6d4a069fdd4acf4d46133e56bfa79cb6d03b77c 172.17.0.2:6379
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
M: e2737cdc7073c3750ffb670bc618dc28cc939877 172.17.0.4:6381
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
S: 30e652ae7137b28a304a6954acc7dd99a070be08 172.17.0.5:16379
   slots: (0 slots) slave
   replicates b6d4a069fdd4acf4d46133e56bfa79cb6d03b77c
S: 808b97759fd358611f94edca21836c4d68d63467 172.17.0.6:16380
   slots: (0 slots) slave
   replicates 8ba84e5b7c76e8530529b41d8c6ee59f3d3edf0c
S: fc156444e5af0817468c95f8a86ec55310e06aed 172.17.0.7:16381
   slots: (0 slots) slave
   replicates e2737cdc7073c3750ffb670bc618dc28cc939877
M: 8ba84e5b7c76e8530529b41d8c6ee59f3d3edf0c 172.17.0.3:6380
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

#### 在cluster中读写数据
```sh
# 进入clusterSlave1容器，然后执行redis-cli -p 16379 
root@5d2ecf9131f7:/data# redis-cli -p 16379
127.0.0.1:16379> set name farb
# 写入数据时发现报错了，name应该放到哈希槽编号5798的主节点，该主节点的ip地址是172.17.0.3，端口是6380，但是一般来说用户希望透明地进行读写而不报错
(error) MOVED 5798 172.17.0.3:6380
127.0.0.1:16379> exit
# 添加-c参数，表示开启集群模式，然后执行set name farb，发现写入成功
root@5d2ecf9131f7:/data# redis-cli -p 16379 -c
127.0.0.1:16379> set name farb
-> Redirected to slot [5798] located at 172.17.0.3:6380
OK
172.17.0.3:6380> get name
"farb"

# 以上可以看到，redis-cli -c参数，在读或写时会自动地把数据定位到正确的服务器上，这种“自动定位”带来的“读写透明”效果正是开发项目所需要的。

```
#### 模拟扩容和数据迁移动作
上面搭建的是3主3从的集群，键的读写会均摊到3个主节点上，cluster集群能很好地应对高并发的挑战。随着业务量的增大，对cluster集群的访问压力可能会增大，此时就需要对集群新增节点来承受更大的并发量。

现在模拟扩容和数据迁移动作，扩容集群，添加一个主节点clusterMaster4和从节点clusterSlave4，然后把数据迁移到新节点上。

##### 1.新增主节点clusterMaster4
```sh
# D:\ArchitectPracticer\Redis\RedisConfCluster目录下创建clusterMaster4.conf文件，内容如下
port 6382
dir /redisConfig
logfile clusterMaster4.log
cluster-enabled yes
cluster-config-file nodes-clusterMaster4.conf
cluster-require-full-coverage no
appendonly yes
appendfsync everysec
cluster-slave-validity-factor 0

# 创建并运行容器
docker run -itd -p 6382:6382 --name clusterMaster4 -v D:\ArchitectPracticer\Redis\RedisConfCluster:/redisConfig:rw redis redis-server /redisConfig/clusterMaster4.conf

# 查看容器IP
PS D:\code\blogs\farb.github.io> docker inspect -f "{{.NetworkSettings.IPAddress}}" clusterMaster4
172.17.0.8
```

##### 2.新增从节点clusterSlave4
```sh
# D:\ArchitectPracticer\Redis\RedisConfCluster目录下创建clusterSlave4.conf文件，内容如下

port 16382
dir /redisConfig
logfile clusterSlave4.log
cluster-enabled yes
cluster-config-file nodes-clusterSlave4.conf
cluster-require-full-coverage no
appendonly yes
appendfsync everysec
cluster-slave-validity-factor 0

# 创建并运行容器
PS D:\code\blogs\farb.github.io> docker run -itd -p 16382:16382 --name clusterSlave4 -v D:\ArchitectPracticer\Redis\RedisConfCluster:/redisConfig:rw redis redis-server /redisConfig/clusterSlave4.conf

# 查看容器IP
PS D:\code\blogs\farb.github.io> docker inspect -f "{{.NetworkSettings.IPAddress}}" clusterSlave4
172.17.0.9
```

##### 3. 使用--cluster add-node命令将新节点加入到集群中
**先看一下redis-cli --cluster中的帮助信息**
```sh
root@fb368ad5ad39:/data# redis-cli --cluster help  
Cluster Manager Commands:
  create         host1:port1 ... hostN:portN
                 --cluster-replicas <arg>
  check          <host:port> or <host> <port> - separated by either colon or space
                 --cluster-search-multiple-owners
  info           <host:port> or <host> <port> - separated by either colon or space
  fix            <host:port> or <host> <port> - separated by either colon or space
                 --cluster-search-multiple-owners
                 --cluster-fix-with-unreachable-masters
  reshard        <host:port> or <host> <port> - separated by either colon or space
                 --cluster-from <arg>
                 --cluster-to <arg>
                 --cluster-slots <arg>
                 --cluster-yes
                 --cluster-timeout <arg>
                 --cluster-pipeline <arg>
                 --cluster-replace
  rebalance      <host:port> or <host> <port> - separated by either colon or space
                 --cluster-weight <node1=w1...nodeN=wN>
                 --cluster-use-empty-masters
                 --cluster-timeout <arg>
                 --cluster-simulate
                 --cluster-pipeline <arg>
                 --cluster-threshold <arg>
                 --cluster-replace

# 添加新节点，如果是添加主节点，只需指定新节点的IP和端口，后面跟上集群中的一个节点的IP和端口即可。
# 如果是添加从节点，需要指定新节点的IP和端口，后面跟上集群中的一个节点的IP和端口，并且指定--cluster-slave和--cluster-master-id 两个参数            
  add-node       new_host:new_port existing_host:existing_port
                 --cluster-slave
                 --cluster-master-id <arg>
  del-node       host:port node_id
  call           host:port command arg arg .. arg
                 --cluster-only-masters
                 --cluster-only-replicas
  set-timeout    host:port milliseconds
  import         host:port
                 --cluster-from <arg>
                 --cluster-from-user <arg>
                 --cluster-from-pass <arg>
                 --cluster-from-askpass
                 --cluster-copy
                 --cluster-replace
  backup         host:port backup_directory
  help

For check, fix, reshard, del-node, set-timeout, info, rebalance, call, import, backup you can specify the host and port of any working node in the cluster.

Cluster Manager Options:
  --cluster-yes  Automatic yes to cluster commands prompts
```
```sh
# 进入集群中的一个节点容器，连接redis-cli,将扩容的主节点加入集群中
root@fb368ad5ad39:/data# redis-cli --cluster add-node 172.17.0.8:6382 172.17.0.2:6379
>>> Adding node 172.17.0.8:6382 to cluster 172.17.0.2:6379
>>> Performing Cluster Check (using node 172.17.0.2:6379)
M: 83df1063a9726b8cbe75c3a838674a8292a8b258 172.17.0.2:6379
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
M: 7983ea1ae620a0de08edf3033f3e9b73c5b98f70 172.17.0.3:6380
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
S: bd33358aa01625c99b7ea10832aa0340ded8816b 172.17.0.6:16380
   slots: (0 slots) slave
   replicates 7983ea1ae620a0de08edf3033f3e9b73c5b98f70
M: 59593191f51ae43efd1663b0d045aaca0dbd0a1e 172.17.0.4:6381
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
S: f2485056c3a66327274235a350cb23257931c0ac 172.17.0.5:16379
   slots: (0 slots) slave
   replicates 83df1063a9726b8cbe75c3a838674a8292a8b258
S: ea1fec8d29809b8b68d2e38c300d9fc667fb7204 172.17.0.7:16381
   slots: (0 slots) slave
   replicates 59593191f51ae43efd1663b0d045aaca0dbd0a1e
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
>>> Getting functions from cluster
>>> Send FUNCTION LIST to 172.17.0.8:6382 to verify there is no functions in it
>>> Send FUNCTION RESTORE to 172.17.0.8:6382
>>> Send CLUSTER MEET to node 172.17.0.8:6382 to make it join the cluster.
[OK] New node added correctly.
root@fb368ad5ad39:/data# redis-cli 
127.0.0.1:6379> cluster nodes
e2737cdc7073c3750ffb670bc618dc28cc939877 172.17.0.4:6381@16381 master - 0 1723300014894 3 connected 10923-16383
30e652ae7137b28a304a6954acc7dd99a070be08 172.17.0.5:16379@26379 slave b6d4a069fdd4acf4d46133e56bfa79cb6d03b77c 0 1723300013891 1 connected
808b97759fd358611f94edca21836c4d68d63467 172.17.0.6:16380@26380 slave 8ba84e5b7c76e8530529b41d8c6ee59f3d3edf0c 0 1723300012887 2 connected
fc156444e5af0817468c95f8a86ec55310e06aed 172.17.0.7:16381@26381 slave e2737cdc7073c3750ffb670bc618dc28cc939877 0 1723300012000 3 connected
b6d4a069fdd4acf4d46133e56bfa79cb6d03b77c 172.17.0.2:6379@16379 myself,master - 0 1723300011000 1 connected 0-5460
8ba84e5b7c76e8530529b41d8c6ee59f3d3edf0c 172.17.0.3:6380@16380 master - 0 1723300014000 2 connected 5461-10922
8ae631285a8532aaeb324b404ab48352a25658ff 172.17.0.8:6382@16382 master - 0 1723300015897 0 connected

# 可以看到新增的主节点已经加入到集群中了
127.0.0.1:6379> cluster nodes
e2737cdc7073c3750ffb670bc618dc28cc939877 172.17.0.4:6381@16381 master - 0 1723300014894 3 connected 10923-16383
30e652ae7137b28a304a6954acc7dd99a070be08 172.17.0.5:16379@26379 slave b6d4a069fdd4acf4d46133e56bfa79cb6d03b77c 0 1723300013891 1 connected
808b97759fd358611f94edca21836c4d68d63467 172.17.0.6:16380@26380 slave 8ba84e5b7c76e8530529b41d8c6ee59f3d3edf0c 0 1723300012887 2 connected
fc156444e5af0817468c95f8a86ec55310e06aed 172.17.0.7:16381@26381 slave e2737cdc7073c3750ffb670bc618dc28cc939877 0 1723300012000 3 connected
b6d4a069fdd4acf4d46133e56bfa79cb6d03b77c 172.17.0.2:6379@16379 myself,master - 0 1723300011000 1 connected 0-5460
8ba84e5b7c76e8530529b41d8c6ee59f3d3edf0c 172.17.0.3:6380@16380 master - 0 1723300014000 2 connected 5461-10922
8ae631285a8532aaeb324b404ab48352a25658ff 172.17.0.8:6382@16382 master - 0 1723300015897 0 connected

# 继续添加从节点
root@fb368ad5ad39:/data# redis-cli --cluster add-node 172.17.0.9:16382 172.17.0.2:6379 --cluster-slave --cluster-master-id 8ae631285a8532aaeb324b404ab48352a25658ff
root@fb368ad5ad39:/data# redis-cli --cluster add-node 172.17.0.9:16382 172.17.0.2:6379 --cluster-slave --cluster-master-id 8ae631285a8532aaeb324b404ab48352a25658ff
>>> Adding node 172.17.0.9:16382 to cluster 172.17.0.2:6379
>>> Performing Cluster Check (using node 172.17.0.2:6379)
M: b6d4a069fdd4acf4d46133e56bfa79cb6d03b77c 172.17.0.2:6379
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
M: e2737cdc7073c3750ffb670bc618dc28cc939877 172.17.0.4:6381
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
S: 30e652ae7137b28a304a6954acc7dd99a070be08 172.17.0.5:16379
   slots: (0 slots) slave
   replicates b6d4a069fdd4acf4d46133e56bfa79cb6d03b77c
S: 808b97759fd358611f94edca21836c4d68d63467 172.17.0.6:16380
   slots: (0 slots) slave
   replicates 8ba84e5b7c76e8530529b41d8c6ee59f3d3edf0c
S: fc156444e5af0817468c95f8a86ec55310e06aed 172.17.0.7:16381
   slots: (0 slots) slave
   replicates e2737cdc7073c3750ffb670bc618dc28cc939877
M: 8ba84e5b7c76e8530529b41d8c6ee59f3d3edf0c 172.17.0.3:6380
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
M: 8ae631285a8532aaeb324b404ab48352a25658ff 172.17.0.8:6382
   slots: (0 slots) master
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
>>> Send CLUSTER MEET to node 172.17.0.9:16382 to make it join the cluster.
Waiting for the cluster to join

>>> Configure node as replica of 172.17.0.8:6382.
[OK] New node added correctly.
# 从节点添加到集群成功

127.0.0.1:6379> cluster nodes
e2737cdc7073c3750ffb670bc618dc28cc939877 172.17.0.4:6381@16381 master - 0 1723300184000 3 connected 10923-16383
30e652ae7137b28a304a6954acc7dd99a070be08 172.17.0.5:16379@26379 slave b6d4a069fdd4acf4d46133e56bfa79cb6d03b77c 0 1723300185865 1 connected
25a08774618e7964f9324e386ce21a2af61bbb9a 172.17.0.9:16382@26382 slave 8ae631285a8532aaeb324b404ab48352a25658ff 0 1723300186869 0 connected
808b97759fd358611f94edca21836c4d68d63467 172.17.0.6:16380@26380 slave 8ba84e5b7c76e8530529b41d8c6ee59f3d3edf0c 0 1723300184863 2 connected
fc156444e5af0817468c95f8a86ec55310e06aed 172.17.0.7:16381@26381 slave e2737cdc7073c3750ffb670bc618dc28cc939877 0 1723300184000 3 connected
b6d4a069fdd4acf4d46133e56bfa79cb6d03b77c 172.17.0.2:6379@16379 myself,master - 0 1723300185000 1 connected 0-5460
8ba84e5b7c76e8530529b41d8c6ee59f3d3edf0c 172.17.0.3:6380@16380 master - 0 1723300186000 2 connected 5461-10922
8ae631285a8532aaeb324b404ab48352a25658ff 172.17.0.8:6382@16382 master - 0 1723300183000 0 connected

# 可以看到新增的主从节点都已经加入到集群中了，但是扩容的主节点现在还并没有分配哈希槽,因而不能保存数据
# 下面进行重新分片,有两种方法，一种是使用交互式命令窗口按照提示操作，第二种是指定redis-cli reshard 命令的其他多个参数 --cluster-from --cluster-to --cluster-slots。这里使用第一种

root@fb368ad5ad39:/data# redis-cli --cluster reshard 172.17.0.8:6382
>>> Performing Cluster Check (using node 172.17.0.8:6382)
M: 8ae631285a8532aaeb324b404ab48352a25658ff 172.17.0.8:6382
   slots: (0 slots) master
   1 additional replica(s)
M: e2737cdc7073c3750ffb670bc618dc28cc939877 172.17.0.4:6381
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
S: 808b97759fd358611f94edca21836c4d68d63467 172.17.0.6:16380
   slots: (0 slots) slave
   replicates 8ba84e5b7c76e8530529b41d8c6ee59f3d3edf0c
M: 8ba84e5b7c76e8530529b41d8c6ee59f3d3edf0c 172.17.0.3:6380
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
S: 30e652ae7137b28a304a6954acc7dd99a070be08 172.17.0.5:16379
   slots: (0 slots) slave
   replicates b6d4a069fdd4acf4d46133e56bfa79cb6d03b77c
S: fc156444e5af0817468c95f8a86ec55310e06aed 172.17.0.7:16381
   slots: (0 slots) slave
   replicates e2737cdc7073c3750ffb670bc618dc28cc939877
M: b6d4a069fdd4acf4d46133e56bfa79cb6d03b77c 172.17.0.2:6379
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
S: 25a08774618e7964f9324e386ce21a2af61bbb9a 172.17.0.9:16382
   slots: (0 slots) slave
   replicates 8ae631285a8532aaeb324b404ab48352a25658ff
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
# 这里询问要移动多少哈希槽？为了平均分配，这里输入4096(16384/4=4096)后回车。
How many slots do you want to move (from 1 to 16384)?4096
# 这里询问接收的节点Id是什么？（也就是要给哪个节点分配哈希槽，可以通过cluster nodes命令查看）
What is the receiving node ID?8ae631285a8532aaeb324b404ab48352a25658ff
# 这一步需要确认要从哪个节点移动哈希槽，输入all，要从所有节点移动哈希槽，目的是保证平均分配，当然你也可以自己指定节点的Id
Please enter all the source node IDs.
  Type 'all' to use all the nodes as source nodes for the hash slots.
  Type 'done' once you entered all the source nodes IDs.
Source node #1: all
    ...
    Moving slot 351 from b6d4a069fdd4acf4d46133e56bfa79cb6d03b77c
    ...
    Moving slot 1361 from b6d4a069fdd4acf4d46133e56bfa79cb6d03b77c
    Moving slot 1362 from b6d4a069fdd4acf4d46133e56bfa79cb6d03b77c
    Moving slot 1363 from b6d4a069fdd4acf4d46133e56bfa79cb6d03b77c
    Moving slot 1364 from b6d4a069fdd4acf4d46133e56bfa79cb6d03b77c
# 上面会显示从哪些节点移动哪些哈希槽，问你要不要继续提议的分片计划，输入yes回车 
Do you want to proceed with the proposed reshard plan (yes/no)?yes

# 然后会输出大量日志，且耗时较长，如果redis中的数据量更大，耗时会更长
# 因此生产环境应该在专门的维护时间段进行扩容升级
...
Moving slot 1364 from 172.17.0.2:6379 to 172.17.0.8:6382: 

# 然后使用redis-cli --cluster check 172.17.0.8:6382命令检查集群状态，发现4个主节点都分配了4096个哈希槽
root@fb368ad5ad39:/data# redis-cli --cluster check 172.17.0.8:6382
172.17.0.8:6382 (8ae63128...) -> 0 keys | 4096 slots | 1 slaves.
172.17.0.4:6381 (e2737cdc...) -> 0 keys | 4096 slots | 1 slaves.
172.17.0.3:6380 (8ba84e5b...) -> 0 keys | 4096 slots | 1 slaves.
172.17.0.2:6379 (b6d4a069...) -> 0 keys | 4096 slots | 1 slaves.
[OK] 0 keys in 4 masters.
0.00 keys per slot on average.
>>> Performing Cluster Check (using node 172.17.0.8:6382)
M: 8ae631285a8532aaeb324b404ab48352a25658ff 172.17.0.8:6382
   slots:[0-1364],[5461-6826],[10923-12287] (4096 slots) master
   1 additional replica(s)
M: e2737cdc7073c3750ffb670bc618dc28cc939877 172.17.0.4:6381
   slots:[12288-16383] (4096 slots) master
   1 additional replica(s)
S: 808b97759fd358611f94edca21836c4d68d63467 172.17.0.6:16380
   slots: (0 slots) slave
   replicates 8ba84e5b7c76e8530529b41d8c6ee59f3d3edf0c
M: 8ba84e5b7c76e8530529b41d8c6ee59f3d3edf0c 172.17.0.3:6380
   slots:[6827-10922] (4096 slots) master
   1 additional replica(s)
S: 30e652ae7137b28a304a6954acc7dd99a070be08 172.17.0.5:16379
   slots: (0 slots) slave
   replicates b6d4a069fdd4acf4d46133e56bfa79cb6d03b77c
S: fc156444e5af0817468c95f8a86ec55310e06aed 172.17.0.7:16381
   slots: (0 slots) slave
   replicates e2737cdc7073c3750ffb670bc618dc28cc939877
M: b6d4a069fdd4acf4d46133e56bfa79cb6d03b77c 172.17.0.2:6379
   slots:[1365-5460] (4096 slots) master
   1 additional replica(s)
S: 25a08774618e7964f9324e386ce21a2af61bbb9a 172.17.0.9:16382
   slots: (0 slots) slave
   replicates 8ae631285a8532aaeb324b404ab48352a25658ff
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.

# 第二种方法，172.17.0.8:6382指执行分配命令的节点， --cluster-slots N 指从每个from节点分配N个哈希槽
redis-cli --cluster reshard 172.17.0.8:6382 --cluster-from e2737cdc7073c3750ffb670bc618dc28cc939877,8ba84e5b7c76e8530529b41d8c6ee59f3d3edf0c,b6d4a069fdd4acf4d46133e56bfa79cb6d03b77c --cluster-to 8ae631285a8532aaeb324b404ab48352a25658ff --cluster-slots 1024
```
#### 故障自动恢复
```sh
root@fb368ad5ad39:/data# redis-cli -c
127.0.0.1:6379> get name
-> Redirected to slot [5798] located at 172.17.0.8:6382
(nil)
172.17.0.8:6382> set name farb
OK
172.17.0.8:6382> get name
"farb"
# 这里模拟节点故障，关闭节点172.17.0.8:6382
172.17.0.8:6382> shutdown
(10.02s)
not connected> exit
root@fb368ad5ad39:/data# redis-cli -p 6379
127.0.0.1:6379> get name
(error) CLUSTERDOWN The cluster is down
127.0.0.1:6379> exit
root@fb368ad5ad39:/data# redis-cli
127.0.0.1:6379> cluster info
cluster_state:fail
cluster_slots_assigned:16384
cluster_slots_ok:12288

# 经过本人大量实验，需要设置clusterMaster4和clusterSlave4配置 cluster-slave-validity-factor 0 ，才可以将从节点提升为主节点
# 可以看到172.17.0.8:6382@16382 master,fail，之前的从节点提升为主节点172.17.0.9:16382@26382 master
127.0.0.1:6379> cluster nodes
808b97759fd358611f94edca21836c4d68d63467 172.17.0.6:16380@26380 slave 8ba84e5b7c76e8530529b41d8c6ee59f3d3edf0c 0 1723389975343 2 connected
fc156444e5af0817468c95f8a86ec55310e06aed 172.17.0.7:16381@26381 slave e2737cdc7073c3750ffb670bc618dc28cc939877 0 1723389977352 3 connected
e2737cdc7073c3750ffb670bc618dc28cc939877 172.17.0.4:6381@16381 master - 0 1723389976348 3 connected 12288-16383
30e652ae7137b28a304a6954acc7dd99a070be08 172.17.0.5:16379@26379 slave b6d4a069fdd4acf4d46133e56bfa79cb6d03b77c 0 1723389974000 1 connected
8ae631285a8532aaeb324b404ab48352a25658ff 172.17.0.8:6382@16382 master,fail - 1723389941225 1723389935203 7 connected
8ba84e5b7c76e8530529b41d8c6ee59f3d3edf0c 172.17.0.3:6380@16380 master - 0 1723389975000 2 connected 6827-10922
b6d4a069fdd4acf4d46133e56bfa79cb6d03b77c 172.17.0.2:6379@16379 myself,master - 0 1723389975000 1 connected 1365-5460
25a08774618e7964f9324e386ce21a2af61bbb9a 172.17.0.9:16382@26382 master - 0 1723389974340 9 connected 0-1364 5461-6826 10923-12287

# 此时再次get name，可以看到成功从新的主节点172.17.0.9:16382得到数据
127.0.0.1:6379> get name
-> Redirected to slot [5798] located at 172.17.0.9:16382
"farb"


# 再次重启旧的主节点，发现旧的主节点已经变成新的主节点的从节点
172.17.0.9:16382> cluster nodes
808b97759fd358611f94edca21836c4d68d63467 172.17.0.6:16380@26380 slave 8ba84e5b7c76e8530529b41d8c6ee59f3d3edf0c 0 1723478740183 2 connected
b6d4a069fdd4acf4d46133e56bfa79cb6d03b77c 172.17.0.2:6379@16379 master - 0 1723478739000 1 connected 1365-5460
8ba84e5b7c76e8530529b41d8c6ee59f3d3edf0c 172.17.0.3:6380@16380 master - 0 1723478742189 2 connected 6827-10922
25a08774618e7964f9324e386ce21a2af61bbb9a 172.17.0.9:16382@26382 myself,master - 0 1723478738000 13 connected 0-1364 5461-6826 10923-12287
8ae631285a8532aaeb324b404ab48352a25658ff 172.17.0.8:6382@16382 slave 25a08774618e7964f9324e386ce21a2af61bbb9a 0 1723478738119 13 connected
fc156444e5af0817468c95f8a86ec55310e06aed 172.17.0.7:16381@26381 slave e2737cdc7073c3750ffb670bc618dc28cc939877 0 1723478741186 3 connected
30e652ae7137b28a304a6954acc7dd99a070be08 172.17.0.5:16379@26379 slave b6d4a069fdd4acf4d46133e56bfa79cb6d03b77c 0 1723478741000 1 connected
e2737cdc7073c3750ffb670bc618dc28cc939877 172.17.0.4:6381@16381 master - 0 1723478738000 3 connected 12288-16383
```

#### 从节点重新恢复为主节点
**原先的主节点宕机后，变成从节点，现在重新恢复为主节点**
```sh
# 进入该从节点的容器，连接redis-cli,执行cluster failover
root@83fd72d8d210:/data# redis-cli -p 6382 -h 172.17.0.8
172.17.0.8:6382> cluster failover takeover
OK
``` 

#### 缩容：删除节点
**只允许删除从节点和空的主节点，如果主节点分配了哈希槽，则需要先将哈希槽转移到其他节点，再进行删除**

```sh
# 如果直接删除分配有哈希槽的主节点，则会报错，提示需要先重新分片
root@fb368ad5ad39:/data# redis-cli -p 6379 --cluster del-node 172.17.0.9:16382  25a08774618e7964f9324e386ce21a2af61bbb9a
>>> Removing node 25a08774618e7964f9324e386ce21a2af61bbb9a from cluster 172.17.0.9:16382
[ERR] Node 172.17.0.9:16382 is not empty! Reshard data away and try again.

redis-cli --cluster reshard 172.17.0.2:6379 --cluster-from 8ae631285a8532aaeb324b404ab48352a25658ff --cluster-to b6d4a069fdd4acf4d46133e56bfa79cb6d03b77c --cluster-slots 4096
......
    Moving slot 12286 from 8ae631285a8532aaeb324b404ab48352a25658ff
    Moving slot 12287 from 8ae631285a8532aaeb324b404ab48352a25658ff
Do you want to proceed with the proposed reshard plan (yes/no)? yes
......
Moving slot 12286 from 172.17.0.8:6382 to 172.17.0.2:6379: 
Moving slot 12287 from 172.17.0.8:6382 to 172.17.0.2:6379: 

# 可以看到172.17.0.8:6382和172.17.0.9:16382都是从节点了
127.0.0.1:6379> cluster nodes
fc156444e5af0817468c95f8a86ec55310e06aed 172.17.0.7:16381@26381 slave e2737cdc7073c3750ffb670bc618dc28cc939877 0 1723562047844 3 connected
25a08774618e7964f9324e386ce21a2af61bbb9a 172.17.0.9:16382@26382 slave b6d4a069fdd4acf4d46133e56bfa79cb6d03b77c 0 1723562045834 17 connected
30e652ae7137b28a304a6954acc7dd99a070be08 172.17.0.5:16379@26379 slave b6d4a069fdd4acf4d46133e56bfa79cb6d03b77c 0 1723562046000 17 connected
8ba84e5b7c76e8530529b41d8c6ee59f3d3edf0c 172.17.0.3:6380@16380 master - 0 1723562049853 2 connected 6827-10922
808b97759fd358611f94edca21836c4d68d63467 172.17.0.6:16380@26380 slave 8ba84e5b7c76e8530529b41d8c6ee59f3d3edf0c 0 1723562046000 2 connected
8ae631285a8532aaeb324b404ab48352a25658ff 172.17.0.8:6382@16382 slave b6d4a069fdd4acf4d46133e56bfa79cb6d03b77c 0 1723562048848 17 connected
e2737cdc7073c3750ffb670bc618dc28cc939877 172.17.0.4:6381@16381 master - 0 1723562048000 3 connected 12288-16383
b6d4a069fdd4acf4d46133e56bfa79cb6d03b77c 172.17.0.2:6379@16379 myself,master - 0 1723562049000 17 connected 0-6826 10923-12287

# 执行删除节点操作
root@fb368ad5ad39:/data# redis-cli -p 6379 --cluster del-node 172.17.0.9:16382  25a08774618e7964f9324e386ce21a2af61bbb9a
>>> Removing node 25a08774618e7964f9324e386ce21a2af61bbb9a from cluster 172.17.0.9:16382
>>> Sending CLUSTER FORGET messages to the cluster...
>>> Sending CLUSTER RESET SOFT to the deleted node.
root@fb368ad5ad39:/data# redis-cli -p 6379 --cluster del-node 172.17.0.2:6379 8ae631285a8532aaeb324b404ab48352a25658ff 
>>> Removing node 8ae631285a8532aaeb324b404ab48352a25658ff from cluster 172.17.0.2:6379
>>> Sending CLUSTER FORGET messages to the cluster...
>>> Sending CLUSTER RESET SOFT to the deleted node.

# 此时集群中又是三主三从了。
127.0.0.1:6379> cluster nodes
fc156444e5af0817468c95f8a86ec55310e06aed 172.17.0.7:16381@26381 slave e2737cdc7073c3750ffb670bc618dc28cc939877 0 1723562354046 3 connected
30e652ae7137b28a304a6954acc7dd99a070be08 172.17.0.5:16379@26379 slave b6d4a069fdd4acf4d46133e56bfa79cb6d03b77c 0 1723562356053 17 connected
8ba84e5b7c76e8530529b41d8c6ee59f3d3edf0c 172.17.0.3:6380@16380 master - 0 1723562353000 2 connected 6827-10922
808b97759fd358611f94edca21836c4d68d63467 172.17.0.6:16380@26380 slave 8ba84e5b7c76e8530529b41d8c6ee59f3d3edf0c 0 1723562354000 2 connected
e2737cdc7073c3750ffb670bc618dc28cc939877 172.17.0.4:6381@16381 master - 0 1723562355050 3 connected 12288-16383
b6d4a069fdd4acf4d46133e56bfa79cb6d03b77c 172.17.0.2:6379@16379 myself,master - 0 1723562354000 17 connected 0-6826 10923-12287
```

#### cluster集群中的常用配置
1. cluster-enabled yes：开启cluster集群模式
2. cluster-config-file：cluster集群节点配置文件，Redis服务器第一次以cluster节点身份启动时自动生成，保存了cluster集群里本节点和其他节点之间的关联方式
3. dir: 指定cluster-config-file文件和日志文件保存的路径
4. logfile: 指定当前节点的日志文件名
5. cluster-node-timeout：cluster集群节点之间的超时时间，单位为毫秒，如果超过该时间，则认为节点已经离线，会向对应的从节点进行故障转移操作。
6. cluster-require-full-coverage：默认为yes。表示cluster集群中有节点失效时该集群是否继续对外提供写服务，出于容错性考虑，建议设置为no,如果设置为yes,那么集群中有节点失效时，该集群只提供只读服务。
7. cluster-migration-barrier：用来设置主节点的最小从节点数量。假设该值为1，当某主节点的从节点数量小于1时，就会从其他从节点个数大于1的主节点那边调剂1个从节点过来，这样做的目的是避免出现不包含从节点的主节点，因为一旦出现这种情况，当主节点失效时，就无法再用从节点进行故障恢复的动作。
8. cluster-slave-validity-factor: 如果设置成０，则无论从节点与主节点失联多久，从节点都会尝试升级成主节点。如果设置成正数，则cluster-node-timeout乘以cluster-slave-validity-factor得到的时间，是从节点与主节点失联后，此从节点数据有效的最长时间，超过这个时间，从节点不会启动故障迁移。假设cluster-node-timeout=5，cluster-slave-validity-factor=10，则如果从节点跟主节点失联超过50秒，此从节点不能成为主节点。注意，如果此参数配置为非0，将可能出现由于某主节点失联却没有从节点能顶上的情况，从而导致集群不能正常工作，在这种情况下，只有等到原来的主节点重新回归到集群，集群才恢复运作。
9. repl-diskless-load : mastar 节点有两种方式传输 RDB，slave 节点也有两种方式加载 master 传输过来的 RDB 数据。
传统方式：接受到数据后，先持久化到磁盘，再从磁盘加载 RDB 文件恢复数据到内存中，这是传统方式。
diskless-load：从 Socket 中一边接受数据，一边解析，实现无盘化。
一共有三个取值可配置：
disabled：不使用 diskless-load 方式，即采用磁盘化的传统方式。
on-empty-db：安全模式下使用 diskless-load（也就 slave 节点数据库为空的时候使用 diskless-load）。
swapdb：使用 diskless-load 方式加载，slave 节点会缓存一份当前数据库的数据，再清空数据库，接着进行 Socket 读取实现加载。缓存一份数据的目的是防止读取 Socket 失败。

#### 补充：使用cluster meet 
**我也实践了下cluster meet命令，但是没有得到预期的结果，不知道是Redis版本（7.2.5）问题还是啥原因。出现的问题是：三主三从的集群可以搭建成功，但是在扩容时，新加入的节点始终是从节点，下面是我的原始命令**

```sh
# 注意这里meet之后的ip是容器所在的IP地址，不是localhost
root@fb368ad5ad39:/data# redis-cli -p 6379 cluster meet 172.17.0.3 6380
OK
root@fb368ad5ad39:/data# redis-cli -p 6379 cluster meet 172.17.0.4 6381
OK
root@fb368ad5ad39:/data# redis-cli -p 6379 cluster meet 172.17.0.5 16379
OK
root@fb368ad5ad39:/data# redis-cli -p 6379 cluster meet 172.17.0.6 16380
OK
root@fb368ad5ad39:/data# redis-cli -p 6379 cluster meet 172.17.0.7 16381
OK

# 一次只能添加一个哈希槽
redis-cli -h hostIp -p port Cluster AddSlots N

# 添加哈希槽范围，依次可以添加多个哈希槽
redis-cli -h hostIp -p port Cluster ADDSLOTSRANGE <start slot> <end slot> [<start slot> <end slot> ...]
    Assign slots which are between <start-slot> and <end-slot> to current node.
```

```sh
# 给节点hostIp:port分配哈希槽N，其中N是哈希槽的编号，从0到16383  
# 由于每次只能分配一个编号，所以这里写一个bash脚本，循环执行
`redis-cli -h hostIp -p port Cluster AddSlots N`

# D:\ArchitectPracticer\Redis\RedisConfCluster目录下新建SetMaster1HashSlots.sh，内容如下：
# 分配哈希槽[0,5460]给clusterMaster1
for i in $(seq 0 5460)
do
    redis-cli -h 172.17.0.2 -p 6379 Cluster AddSlots $i
done

# 以上脚本等价于
root@fb368ad5ad39:/data# redis-cli -h 172.17.0.2 -p 6379                       
172.17.0.2:6379> cluster addslotsrange 0 5460
OK

# D:\ArchitectPracticer\Redis\RedisConfCluster目录下新建SetMaster2HashSlots.sh，内容如下：
# 分配哈希槽[5461,10922]给clusterMaster2
for i in $(seq 5461 10922)
do
    redis-cli -h 172.17.0.3 -p 6380 Cluster AddSlots $i
done
# 以上脚本等价于
root@fb368ad5ad39:/data# redis-cli -h 172.17.0.3 -p 6380 
172.17.0.3:6380> CLUSTER ADDSLOTSRANGE 5461 10922
OK

# D:\ArchitectPracticer\Redis\RedisConfCluster目录下新建SetMaster3HashSlots.sh，内容如下：
# 分配哈希槽[10923,16383]给clusterMaster3
for i in $(seq 10923 16383)
do
    redis-cli -h 172.17.0.4 -p 6381 Cluster AddSlots $i
done
# 以上脚本等价于
root@fb368ad5ad39:/data# redis-cli -h 172.17.0.4 -p 6381
172.17.0.4:6381> cluster addslotsrange 10923 16383
OK

# 通过exit退出redis-cli，进入容器，然后依次执行脚本，会看到一直打印ok
/redisConfig/SetMaster1HashSlots.sh 
/redisConfig/SetMaster21HashSlots.sh 
/redisConfig/SetMaster3HashSlots.sh 

# 运行完毕后，此时已经给clusterMaster1、clusterMaster2、clusterMaster3分别分配了哈希槽，进入clusterMaster1容器，查看cluster info如下：

127.0.0.1:6379> cluster info
# 集群状态是ok，说明集群已经完成初始化
cluster_state:ok
# 分配的哈希槽的个数为16384
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
# 可以看到集群中有6个节点
cluster_known_nodes:6
# 集群中主节点个数为3
cluster_size:3

```

**关联主节点和从节点**      
设置从节点的方式是用redis-cli命令进入redis服务器从节点，然后运行`cluster replicate masterNodeId`.
masterNodeId是master节点的节点id，可以通过`cluster nodes`命令查看。
```sh
# 进入clusterMaster1容器,连接到redis-cli，然后查看集群节点id,很明显，第一列就是节点Id
127.0.0.1:6379> cluster nodes
94c321bf8580621f267e7a0b502f6c1bd120f203 172.17.0.5:16379@26379 master - 0 1722779543000 3 connected
8b9e6e09898be857fcd33e4a2611c42268ce7b1c 172.17.0.2:6379@16379 myself,master - 0 1722779542000 2 connected 0-5460
207623a24974859ede6b1a59a046cb10fdfb7d54 172.17.0.7:16381@26381 master - 0 1722779544247 5 connected
8794e2d834a7591630dced8cbaa073c1216de978 172.17.0.6:16380@26380 master - 0 1722779544000 4 connected
fbb526d37ce8ef0067387d7810677200b5ed5d89 172.17.0.4:6381@16381 master - 0 1722779544000 0 connected 10923-16383
8192cc1bb0de71879b1f7174676260d3d5510a7f 172.17.0.3:6380@16380 master - 0 1722779543244 1 connected 5461-10922

# 设置clusterMaster1为clusterSlave1的主节点
PS D:\code\blogs\farb.github.io> docker exec -it clusterSlave1 bash
root@5d2ecf9131f7:/data# redis-cli -p 16379
127.0.0.1:16379> cluster replicate 8b9e6e09898be857fcd33e4a2611c42268ce7b1c
OK

# 设置clusterMaster2为clusterSlave2的主节点
PS D:\code\blogs\farb.github.io> docker exec -it clusterSlave2 bash
root@44e2e24b798a:/data# redis-cli -p 16380
127.0.0.1:16380> cluster replicate 8192cc1bb0de71879b1f7174676260d3d5510a7f
OK

# 设置clusterMaster3为clusterSlave3的主节点
PS D:\code\blogs\farb.github.io> docker exec -it clusterSlave3 bash
root@ef29a379b557:/data# redis-cli -p 16381
127.0.0.1:16381> cluster replicate fbb526d37ce8ef0067387d7810677200b5ed5d89
OK

# 至此，cluster集群三主三从节点已经搭建完成。可以进入任意一台服务器进行验证。
127.0.0.1:16381> cluster nodes
8794e2d834a7591630dced8cbaa073c1216de978 172.17.0.6:16380@26380 slave 8192cc1bb0de71879b1f7174676260d3d5510a7f 0 1722780289575 1 connected
fbb526d37ce8ef0067387d7810677200b5ed5d89 172.17.0.4:6381@16381 master - 0 1722780291000 0 connected 10923-16383
207623a24974859ede6b1a59a046cb10fdfb7d54 172.17.0.7:16381@26381 myself,slave fbb526d37ce8ef0067387d7810677200b5ed5d89 0 1722780289000 0 connected
8192cc1bb0de71879b1f7174676260d3d5510a7f 172.17.0.3:6380@16380 master - 0 1722780290586 1 connected 5461-10922
94c321bf8580621f267e7a0b502f6c1bd120f203 172.17.0.5:16379@26379 slave 8b9e6e09898be857fcd33e4a2611c42268ce7b1c 0 1722780291589 2 connected
8b9e6e09898be857fcd33e4a2611c42268ce7b1c 172.17.0.2:6379@16379 master - 0 1722780290000 2 connected 0-5460

```
**使用cluster meet命令创建集群之后，不论是使用cluster meet还是redis-cli --cluster add-node添加的节点都是从节点。具体原因待以后继续调查。**