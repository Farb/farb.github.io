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

### 地理位置相关操作
#### 用geoadd命令存储地理位置
已知上海的经纬度是东经120°52′到122°12′，北纬30°40′到31°53′。

```sh
geoadd key [NX|XX] [CH] longitude latitude member [longitude latitude member ...]

# 分别将上海区域的四个角的经纬度坐标加到key为shanghai的有序集合中
127.0.0.1:6379> geoadd shanghai  120.52 31.53 wn
(integer) 1
127.0.0.1:6379> geoadd shanghai 120.52 30.40 ws
(integer) 1
127.0.0.1:6379> geoadd shanghai 122.12 31.53 en
(integer) 1
127.0.0.1:6379> geoadd shanghai 122.12 30.40 es
```

#### 获取地理位置的经纬度信息
```sh
geopos key [member [member ...]]

127.0.0.1:6379> geopos shanghai
(empty array)
# 获取上海西北角的经纬度，不传member或者member不存在，都没有返回值
127.0.0.1:6379> geopos shanghai wn
1) 1) "120.52000075578689575"
   2) "31.53000103201371473"
127.0.0.1:6379> geopos shanghai notexist
1) (nil)
```

#### 查询指定范围内的地理信息

```sh
georadius key longitude latitude radius M|KM|FT|MI [WITHCOORD] [WITHDIST] [WITHHASH] [COUNT count [ANY]] [ASC|DESC] [STORE key|STOREDIST key]
# longitude latitude是中心点
# radius是半径
# M|KM|FT|MI是单位
# [WITHCOORD]加上该参数会把对应地理信息的经纬度一起返回
# [WITHDIST] 会把和中心点的距离一起返回
# [WITHHASH] 会返回地理信息的哈希编码
# [COUNT count [ANY]] 指定返回数据的数量
# [ASC|DESC] 指定返回的顺序
# [STORE key|STOREDIST key] 会将返回结果或者返回的距离存储到缓存中

# 下面返回key为shanghai的数据集中，距离中心点（上海西南角）200KM的所有点，同时返回点的坐标，距离，并以升序排列
127.0.0.1:6379> georadius shanghai 120.52 30.40 200 KM withcoord withDist asc
1) 1) "ws"
   2) "0.0001"
   3) 1) "120.52000075578689575"
      2) "30.39999952668997452"
2) 1) "wn"
   2) "125.6858"
   3) 1) "120.52000075578689575"
      2) "31.53000103201371473"
3) 1) "es"
   2) "153.4932"
   3) 1) "122.11999744176864624"
      2) "30.39999952668997452"
4) 1) "en"
   2) "197.6902"
   3) 1) "122.11999744176864624"
      2) "31.53000103201371473"
```

#### 查询地理位置间的距离
```sh
 geodist key member1 member2 [M|KM|FT|MI]

# 返回key为shanghai的数据集合中，ws和en的距离，单位为km，
# 可以看到，上海西南到东北的距离大约是198km
# 如果有任何一个坐标名称不存在坐标，则返回nil
 127.0.0.1:6379> geodist shanghai ws en KM
"197.6902"
```

### 位图的应用
和编程语言中的位操作是一个道理，就是二进制运算。
### setbit 和 getbit
```sh
setbit key offset value

# offset表示偏移量
# value表示设置的值，为0或1

 127.0.0.1:6379> setbit myBitmap 0 1
(integer) 0
127.0.0.1:6379> setbit myBitmap 1 1
(integer) 0
127.0.0.1:6379> setbit myBitmap 2 0
(integer) 0
127.0.0.1:6379> setbit myBitmap 3 0
(integer) 0
# 2不在0和1的范围，所以报错
127.0.0.1:6379> setbit myBitmap 5 2
(error) ERR bit is not an integer or out of range
127.0.0.1:6379> getbit myBitmap 1
(integer) 1
127.0.0.1:6379> getbit myBitmap 2
(integer) 0
# 键不存在返回0
127.0.0.1:6379> getbit notExist 5
(integer) 0
```

#### 用bitop对位图进行运算

```sh
# 支持4种逻辑运算，destkey是结果的存储键
bitop AND|OR|XOR|NOT destkey key [key ...]

127.0.0.1:6379> setbit b1 0 1
(integer) 0
127.0.0.1:6379> setbit b1 1 1
(integer) 0
127.0.0.1:6379> setbit b1 3 1
(integer) 0
127.0.0.1:6379> setbit b2 2 1
(integer) 0

# 不小心写错了名字，可以看到结果执行不成功，返回0
127.0.0.1:6379> bitop and res bit1 bit2
(integer) 0
# 1011 & 0100 = 0
127.0.0.1:6379> bitop and res b1 b2
(integer) 1
127.0.0.1:6379> get res
"\x00"

# 1011 | 0100 = 1111
127.0.0.1:6379> bitop or res b1 b2
(integer) 1
127.0.0.1:6379> getbit res 2
(integer) 1

# not 1011 = 0100
127.0.0.1:6379> bitop not res  b1
(integer) 1
127.0.0.1:6379> getbit res 1
(integer) 0
127.0.0.1:6379> getbit res 2
(integer) 1

#1011 xor 0100 = 1111
127.0.0.1:6379> bitop xor res b1 b2
(integer) 1
127.0.0.1:6379> getbit res 0
(integer) 1
127.0.0.1:6379> getbit res 3
(integer) 1
```

#### bitcount操作
**bitcount统计键key的位图中1出现的次数**
```sh
 bitcount key [start end [BYTE|BIT]]
# start end表示统计的字节组范围

127.0.0.1:6379> setbit user1 0 1
(integer) 0
127.0.0.1:6379> setbit user1 3 1
(integer) 0
127.0.0.1:6379> setbit user1 7 1
(integer) 0
# 这个不难理解，结果存储的格式是10001001，一个字节中有3个1
127.0.0.1:6379> bitcount user1
(integer) 3

# 默认单位是BYTE，也就是统计第0到0个字节范围中的1，其实就是第一个字节，结果还是3
127.0.0.1:6379> bitcount user1 0 0
(integer) 3

# 如果单位是BIT，则前6位中的1有2个
127.0.0.1:6379> bitcount user1 0 5 BIT
(integer) 2
```

### 慢查询实战
#### 慢查询相关的配置参数
- slow-log-slower-than: 单位为微秒。即超过该参数指定时间的查询会记录到日志中。
- slowlog-max-len: 慢查询日志里记录的日志条数，当超过该条数时，会删除最老的一条日志。
- 
```sh
# 可以直接修改redis.conf配置文件或者通过命令修改
127.0.0.1:6379> config set slowlog-log-slower-than 1
OK
127.0.0.1:6379> config set slowlog-max-len 100
OK
127.0.0.1:6379> config rewrite
```
#### 用slowlog get命令观察慢查询

```sh
# 为了演示慢查询，上面设置了 config set slowlog-log-slower-than 1
# 即超过1毫秒的命令都会记录到慢日志中
127.0.0.1:6379> slowlog get
1) 1) (integer) 3
   2) (integer) 1721923737
   3) (integer) 3
   4) 1) "get"
      2) "name"
   5) "127.0.0.1:45308"
   6) ""
2) 1) (integer) 2
   2) (integer) 1721923735
   3) (integer) 5
   4) 1) "set"
      2) "name"
      3) "farb"
```
#### 慢查询相关命令
```sh
# 获取指定数量的慢查询日志
slowlog get [count]

127.0.0.1:6379> slowlog get 1
1) 1) (integer) 5
   2) (integer) 1721923903
   3) (integer) 3
   4) 1) "slowlog"
      2) "get"
      3) "3"
   5) "127.0.0.1:45308"
   6) ""

# 获取慢查询的长度
slowlog len

# 清空慢查询日志
slowlog reset

127.0.0.1:6379> slowlog len
(integer) 7

127.0.0.1:6379> slowlog reset
OK
127.0.0.1:6379> slowlog len
(integer) 1
```
