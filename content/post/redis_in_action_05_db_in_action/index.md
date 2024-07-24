---
title: 基于Docker的Redis实战--Redis数据库操作实践
description:
slug: redis_in_action_05_db_in_action
date: 2024-07-24
image: 
categories:
    - Redis
tags:
    - redis
    - docker
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---


### 切换数据库操作
**默认情况下，Redis服务器在启动时会创建16个数据库，不同的应用程序可以连到不同的数据库**
#### 查看和设置默认的数据库个数
```sh
# 启动一个redis-server容器，注意，-v参数是将宿主机的路径映射到docker容器中的路径，路径不能加引号
 PS D:\code\blogs\farb.github.io> docker run -itd --name redis-server -v D:\ArchitectPracticer\Redis\RedisConf\redis.conf:/redisConfig/redis.conf -p 6379:6379 redis:latest redis-server /redisConfig/redis.conf  
6f245e0ac03b486dd05b414ba14a4260c23330074c7faff8ba112798614c1ea4

# 进入容器，并查看数据库的个数，默认为16
PS D:\code\blogs\farb.github.io> docker exec -it redis-server /bin/bash 
root@6f245e0ac03b:/data# redis-cli
127.0.0.1:6379> config get databases
1) "databases"
2) "16"

# 修改宿主机的redis.conf配置文件databases 12,将数据库个数改为12
# 先停止容器，再启动容器(直接重启restart也可以)，再次查看数据库的个数为12
PS D:\code\blogs\farb.github.io> docker stop redis-server
redis-server
PS D:\code\blogs\farb.github.io> docker start redis-server
redis-server
PS D:\code\blogs\farb.github.io> docker exec -it redis-server /bin/bash
root@6f245e0ac03b:/data# redis-cli
127.0.0.1:6379> config get databases
1) "databases"
2) "12"
```
#### 用select命令切换数据库
```sh
# 通过client list命令中的db=0可知，当前客户端用的是0号数据库
127.0.0.1:6379> client list
id=3 addr=127.0.0.1:39034 laddr=127.0.0.1:6379 fd=8 name= age=111 idle=0 flags=N db=0 sub=0 psub=0 ssub=0 multi=-1 qbuf=26 qbuf-free=20448 argv-mem=10 multi-mem=0 rbs=1024 rbp=0 obl=0 oll=0 omem=0 tot-mem=22426 events=r cmd=client|list user=default redir=-1 resp=2 lib-name= lib-ver=

# 在0号数据库设置salary之后，可以获取值
127.0.0.1:6379> set salary 666
OK
127.0.0.1:6379> get salary
"666"

# 当切换到1号数据库时就读取不到
127.0.0.1:6379> select 1
OK
127.0.0.1:6379[1]> get salary
(nil)

# 再次切换回0号数据库时，可以读取到
127.0.0.1:6379[1]> select 0
OK
127.0.0.1:6379> get salary
"666"
```

### Redis事务操作
**事务具有ACID特性，即原子性、一致性、隔离性和持久性。通过事务，可以让一段代码全部执行，或者全都不执行**
#### 事务的概念和ACID特性
A:Atomicity(原子性)，即事务是一个不可分割的实体，事务中的操作要么全部执行，要么全都不执行。
C:Consistency(一致性)，即事务前后数据完整性必须一致。
I:Isolation(隔离性)，一个事务内部操作对其他事务是隔离的，并发执行的各个事务互不干扰。
D:Durability(持久性)，一个事务一旦提交，它对数据库的改变就是持久性的，哪怕数据库出现故障，事务执行后的操作也该丢失。

#### 实现Redis事务的相关命令
**Redis有4个命令和事务有关：**
multi:开启redis事务     
exec:提交事务       
discard:取消事务    
watch:监视指定的键值对，从而让事务有条件地执行

```sh
# 可以看到，开启事务之前和提交事务后，执行命令会立即返回结果
# 开始事务后，命令执行后不直接返回结果，而是返回QUEUED,表示放到了事务队列中。
# 当exec执行后，会一次性地执行事务队列中的命令。
127.0.0.1:6379> set name farb
OK
127.0.0.1:6379> multi
OK
127.0.0.1:6379(TX)> set age 18
QUEUED
127.0.0.1:6379(TX)> set height 180
QUEUED
127.0.0.1:6379(TX)> exec
1) OK
2) OK
127.0.0.1:6379> get age
"18"
```

#### 通过discard命令撤销事务中的操作
```sh
# 执行multi开启事务后，执行的set和get命令进入事务队列，
# 最后取消事务后，get sex返回nil，表示键不存在，说明事务没执行，符合预期。
127.0.0.1:6379> multi
OK
127.0.0.1:6379(TX)> set sex male
QUEUED
127.0.0.1:6379(TX)> get name
QUEUED
127.0.0.1:6379(TX)> discard
OK
127.0.0.1:6379> get sex
(nil)
```

#### Redis持久化和事务持久性
**Redis持久化方式有两种：AOF（Append Only File）和RDB(Redis Database)**
Redis的AOF持久化方式可以确保事务的持久性，而RDB不能，因为RDB的持久化方式需要满足一定的条件才能触发。
如果刚好在没满足触发条件时Redis服务器发生故障，则无法将事务影响的数据持久化到硬盘上，从而下次重启时无法恢复数据，从而导致事务持久性失效。
而AOF每个写命令都会持久化到硬盘，因而可以保证事务的持久性。

``` sh
# 可以通过下面命令实现基于AOF的持久化，并将配置保存到redis.conf配置文件
127.0.0.1:6379> config set appendfsync always
OK
127.0.0.1:6379> config rewrite

# 通过以下方式实现基于RDB的持久化
127.0.0.1:6379> config set dir /
127.0.0.1:6379> config set dbfilename redis.rdb
127.0.0.1:6379> config rewrite
```

#### 用watch命令监视指定键
其他客户端C2修改watch监控的变量，会阻止客户端C1中的事务对该变量的修改，因为客户端C1监控时的值和更新时刻的值已经发生了变化。
```sh
# 在第一个控制台执行以下事务，在执行exec前，在第二个控制台连接新的客户端，更新salary的值为666888
127.0.0.1:6379> watch salary
OK
127.0.0.1:6379> multi
OK
127.0.0.1:6379(TX)> set salary 1000
QUEUED
# 因为在执行exec的时刻，监控的salary已经发生了改变，因而事务执行失败，结果返回nil
127.0.0.1:6379(TX)> exec
(nil)
# 返回的是第二个客户端修改的值
127.0.0.1:6379> get salary
"666888"

# 注意：unwatch只能撤销对所有键的监控，如果指定一个键则会报错
127.0.0.1:6379> unwatch
OK

# 在第二个客户端更新salary
PS D:\code\blogs\farb.github.io> docker exec -it redis-server /bin/bash
root@6f245e0ac03b:/data# redis-cli
127.0.0.1:6379> get salary
"666"
127.0.0.1:6379> set salary 666888
OK
```










































