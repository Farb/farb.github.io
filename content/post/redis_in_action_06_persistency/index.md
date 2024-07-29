---
title: 基于Docker的Redis实战--Redis数据持久化操作
description:
slug: redis_in_action_06_persistency
date: 2024-07-27
image: 
categories:
    - Redis
tags:
    - redis
    - docker
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---


### Redis持久化机制概述
**对于Redis而言，持久化机制是指把内存中的数据存为硬盘文件，当Redis重启或服务器故障时，能根据持久化的硬盘文件恢复数据。Redis有两种持久化方式，分别是AOF(Append Only File,只追加文件)和RDB（Redis database,基于Redis数据库）**
#### 基于AOF的持久化机制
AOF会以日志的方式记录每个Redis的写命令，并在Redis服务器重启时重新执行AOF日志文件中的命令，从而恢复数据。
AOF持久化默认配置关闭，如果打开，每当发生写的命令，该命令就会被记录到AOF缓冲区里，AOF缓冲区会根据事先配置的策略定期与硬盘文件进行同步操作。
当AOF文件大到一定程度后会被重写，在不影响持久化结果的前提下进行压缩。

**AOF重写：随着持久化数据的增多，对应的AOF文件会越来越大，可能会影响性能，此时Redis会创建新的AOF文件替代现有的，在数据恢复时，效果相同，但新文件不会包含冗余命令，所以文件会比原来的小。**
当Redis发生故障重启时，根据AOF日志文件恢复数据的过程如下：
1. 创建一个伪客户端（fake client）,不连接任何网络，除此之外和真实的客户端都一样；
2. 该伪客户端从AOF日志文件中依次读取命令并执行，直到完成所有命令。

AOF的持久化机制具有实时存储的特性，因此可以在读写关键数据时开启。
#### 基于RDB的持久化机制
Redis默认的持久化机制就RDB，RDB的持久化方式会把当前内存中的所有Redis键值对数据以快照的方式保存到硬盘文件，如果需要恢复数据，就把快照文件读取到内存。
RDB快照文件是经压缩的二进制文件，它的存储路径可以在redis.conf中指定，也可以在redis运行时通过命令设置。
Redis的RDB持久化机制触发机制有两种：

1. save和bgsave等命令手动触发；
2. 通过redis.conf配置文件里设置的方式定期把数据写入快照。

基于RDB的持久化方式适合数据备份和灾备场景，但RDB无法实现即使备份，两次备份之间必然会丢数据。

### AOF持久化机制实践
#### AOF配置文件的说明

``` sh
# 开启AOF功能，默认是关闭的
appendonly yes

# 开启aof功能后，通过appendfsync设置持久化策略
appendfsync always/everysec/no
# always: 每次写命令时都会触发持久化动作，这种方式可能影响Redis性能。
# everysec: 每秒触发一次持久化动作，平衡了持久化需求和性能，一般取这个值。
# no: 由操作系统决定持久化的频率，这种性能最好，但每次持久化操作的间隔可能较长，当故障发生时可能会丢失数据。

dir aofDir
# dir指定保存持久化文件的目录

appendfilename aofFileName
# 指定持久化文件的名称，默认文件名为appendonly.aof

auto-load-truncated
# 定义aof文件的加载策略。当AOF持久化文件损坏时启动Redis是否会自动加载，默认为yes

no-appendfsync-on-rewrite yes/no
# 平衡性能和安全性。字面意思是：AOF重写时是否禁用AOF追加。默认为no
# 如果为yes,则会提高重写性能，但可能会丢失写的数据。
# 如果为no，则不会丢失数据，但重写性能可能会降低。

auto-aof-rewrite-percentage
# 指定自动重写的条件。默认是100，即如果当前的AOF文件比上次执行重写时的文件大100%时会再次触发重写操作。如果为0，则不触发。

auto-aof-rewrite-min-size
# 指定自动AOF重写的AOF文件的最低要求大小。默认为64MB 

# 注意： auto-aof-rewrite-percentage 和 auto-aof-rewrite-min-size 必须同时满足条件才会触发。

```

**注意：也可以通过bgrewriteaof命令手动触发AOF重写**

#### 实践AOF持久化
Redis 7 中引入的 Multi-Part AOF (MP-AOF) 是一种改进的持久化机制，旨在优化 AOF (Append Only File) 的性能和效率，尤其是在高并发写入场景下。传统的 AOF 机制在主进程处理客户端请求的同时，还需要负责 AOF 日志的写入和重写，这可能导致性能瓶颈。

MP-AOF 的主要目标是减少主进程的负担，通过将 AOF 日志的写入和重写操作分离到独立的后台线程中，从而提高整体的系统吞吐量。MP-AOF 将 AOF 文件分为两部分：

Base RDB 文件：这部分是数据库的快照，类似于 RDB 文件，用于快速恢复数据集的基本状态。
Incremental AOF 文件：这部分包含自上次快照以来的所有写操作，用于增量更新数据库状态。
appendonly.aof.1.incr.aof 文件就是 MP-AOF 架构下的 Incremental AOF 文件的一个实例。这个文件包含了自上次 RDB 快照之后的所有写入操作。

以下是 MP-AOF 的几个关键特点：

- 分离写操作：MP-AOF 将写操作的记录和重写任务分配给单独的线程，减少了主进程的负载。
- 异步重写：AOF 的重写操作不再阻塞主进程，而是由后台线程异步完成。
- 高效恢复：在服务器启动时，首先加载 Base RDB 文件来快速恢复大部分数据，然后应用 Incremental AOF 文件中的操-作来更新数据集，这种分阶段的恢复过程比传统 AOF 更高效。
- 减少磁盘 I/O：通过减少主进程的磁盘 I/O 操作，提高了系统的整体响应速度。

```sh
# 创建redis-server 容器，使用-v参数将宿主机的路径绑定为配置文件路径，rw指定有对配置目录的读写权限
 docker run -itd --name redis-server -v D:/ArchitectPracticer/Redis/RedisConf:/redisConfig:rw -p 6379:6379 redis:latest redis-server /redisConfig/redis.conf

 # 进入容器并启动redis-cli，执行set和get命令，然后查看AOF的文件
 PS D:\code\blogs\farb.github.io> docker exec -it redis-server bash
root@74551bec3ecc:/data# redis-cli
127.0.0.1:6379> set name farb
OK
127.0.0.1:6379> set age 18
OK
127.0.0.1:6379> set height 180
OK
127.0.0.1:6379> get name
"farb"

# 在Redis 7.2.5中可以看到AOF文件生成了3个，默认在appendonlydir目录。其中incr.aof文件是增量文件，base.rdb文件是基础文件。
PS D:\ArchitectPracticer\Redis\RedisConf\appendonlydir> ls
    目录: D:\ArchitectPracticer\Redis\RedisConf\appendonlydir
Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----         2024/7/28     22:43             88 appendonly.aof.1.base.rdb
-a----         2024/7/28     22:45            120 appendonly.aof.1.incr.aof
-a----         2024/7/28     22:43             88 appendonly.aof.manifest

# 打开appendonly.aof.1.incr.aof，可以看到写入的命令，get命令没有记录。
*2
$6
SELECT
$1
0
*3
$3
set
$4
name
$4
farb
*3
$3
set
$3
age
$2
18
*3
$3
set
$6
height
$3
180
```

#### 观察重写AOF的文件的效果
```sh
# 向list中写入数据
127.0.0.1:6379> rpush namelist alice
(integer) 1
127.0.0.1:6379> rpush namelist bob
(integer) 2
127.0.0.1:6379> rpush namelist candy
(integer) 3

# 观察aof文件如下
rpush
$8
namelist
$3
bob
*3
$5
rpush
$8
namelist
$5
candy
```
**手动执行bgrewriteaof命令，再次查看aof日志文件,可以看到之前的appendonly.aof.1.incr.aof已经删除，而appendonly.aof.2.incr.aof文件生成了，并且是空的，打开appendonly.aof.2.base.rdb可以看到重写的内容。一般不会使用bgrewriteaof命令，而是通过auto-aof-rewrite-percentage和auto-aof-rewrite-min-size两个参数控制AOF重写策略。**

``` sh
PS D:\ArchitectPracticer\Redis\RedisConf\appendonlydir> ls
    目录: D:\ArchitectPracticer\Redis\RedisConf\appendonlydir
Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----         2024/7/29      0:09            132 appendonly.aof.2.base.rdb
-a----         2024/7/29      0:09              0 appendonly.aof.2.incr.aof
-a----         2024/7/29      0:09             88 appendonly.aof.manifest

REDIS0011�	redis-ver7.2.5�
redis-bits�@�ctime�2m�f�used-mem°�namelist�alice�bob�candy

# appendonly.aof.manifest文件内容变成：
file appendonly.aof.2.base.rdb seq 2 type b
file appendonly.aof.2.incr.aof seq 2 type i
# 也就是说这个清单文件记录了base和incr两个文件名，并记录了这两个文件的顺序号和类型。当前是第二个序列，且b是base,i是incr。
```

再次向list中写入一条数据，然后查看aof日志文件
```sh
127.0.0.1:6379> rpush namelist hugh
(integer) 4
127.0.0.1:6379> bgrewriteaof
Background append only file rewriting started

# 观察appendonly.aof.manifest文件如下
file appendonly.aof.3.base.rdb seq 3 type b
file appendonly.aof.3.incr.aof seq 3 type i

# 观察appendonly.aof.3.incr.aof文件为空
# 观察appendonly.aof.3.base.rdb文件如下:
REDIS0011�	redis-ver7.2.5�
redis-bits�@�ctime�C��f�used-mem��T�aof-base��namelist�alice�bob�candy�hugh

```

**经过上面的实践和观察，发现Redis7之前AOF重写都是在同一个文件名中进行的，而Redis7之后，AOF重写是分离到独立的3个文件，appendonly.aof.manifest文件记录了rdb和incr文件的名称、序列号和类型，incr文件记录了写入的命令，而rdb文件记录了重写的内容，且incr和rdb每次重写后序列号都会自增。**

#### 模拟数据恢复的流程
上面redis中已经创建了一个namelist，现在模拟一下数据恢复的流程，先通过flushall命令清空redis数据库，模拟宕机，然后在incr的aof文件中删除最后的flushall命令（如果不删除，恢复数据时会执行flushall,把之前恢复的数据再次清空），然后删除旧的容器redis-server,并重新创建启动redis-server，然后查看namelist，发现namelist已经恢复，说明数据已经恢复。

```sh
# 清空数据库并查看aof文件
127.0.0.1:6379> flushall
OK
127.0.0.1:6379> keys *
(empty array)

# 观察appendonly.aof.3.incr.aof文件如下：
*2
$6
SELECT
$1
0
*1
$8
flushall

# 删除flushall的相关命令并保存文件

# 退出redis-cli
127.0.0.1:6379> exit

# 退出容器
root@f501aa313152:/data# exit
exit

# 强制删除容器
PS D:\code\blogs\farb.github.io> docker rm -f redis-server  

redis-server

# 创建redis-server容器并运行
PS D:\code\blogs\farb.github.io> docker run -itd --name redis-server -p 6379:6379 -v D:\ArchitectPracticer\Redis\RedisConf:/redisConfig:rw redis:latest redis-server /redisConfig/redis.conf
2b939fe1c091990c6755770078c45fb37637ce89c7d6baa441d4d818dc17647d

# 进入redis-server容器，运行bash
PS D:\code\blogs\farb.github.io> docker exec -it redis-server bash

# 进入redis-cli，可以看到之前aof中的数据已经恢复了
root@2b939fe1c091:/data# redis-cli
127.0.0.1:6379> lrange namelist 0 -1
1) "alice"
2) "bob"
3) "candy"
4) "hugh"

```

#### 修复AOF文件
数据恢复时，如果AOF文件损坏，可以通过以下步骤修复AOF文件：
```sh
redis-check-aof [--fix|--truncate-to-timestamp $timestamp] <file.manifest|file.aof>

# redis7之前，直接指定appendonlyfile.aof文件即可；Redis7之后，需要指定appendonly.aof.manifest文件，默认目录为appendonlydir
root@2b939fe1c091:/data# redis-check-aof --fix /redisConfig/appendonlydir/appendonly.aof.manifest
Start checking Multi Part AOF
Start to check BASE AOF (RDB format).
[offset 0] Checking RDB file /redisConfig/appendonlydir/appendonly.aof.3.base.rdb
[offset 26] AUX FIELD redis-ver = '7.2.5'
[offset 40] AUX FIELD redis-bits = '64'
[offset 52] AUX FIELD ctime = '1722262595'
[offset 67] AUX FIELD used-mem = '939216'
[offset 79] AUX FIELD aof-base = '1'
[offset 81] Selecting DB ID 0
[offset 138] Checksum OK
[offset 138] \o/ RDB looks OK! \o/
[info] 1 keys read
[info] 0 expires
[info] 0 already expired
RDB preamble is OK, proceeding with AOF tail...
AOF analyzed: filename=appendonly.aof.3.base.rdb, size=138, ok_up_to=138, ok_up_to_line=1, diff=0
BASE AOF appendonly.aof.3.base.rdb is valid
Start to check INCR files.
INCR AOF appendonly.aof.3.incr.aof is empty
All AOF files and manifest are valid

# 从上面的输出可以看到，先检查rdb是否有效，再检查增量文件是否有效
```
