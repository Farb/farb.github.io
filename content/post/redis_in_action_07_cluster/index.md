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