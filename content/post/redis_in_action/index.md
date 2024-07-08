---
title: 基于Docker的Redis实战
description:
slug: docker-redis-study
date: 2024-06-20
image: 
categories:
    - Redis
tags:
    - redis,docker
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---

## 构建Redis开发环境
下载Docker desktop，配置可用镜像地址。
docker 镜像可用地址： https://www.aabcc.top/archives/m7NPfx1D   
Redis客户端AnotherRedisDesktopManager： https://gitee.com/qishibo/AnotherRedisDesktopManager/releases

### 必要的docker技能
#### docker镜像相关的命令
``` sh
# 查看本机所有镜像
docker images 

# 拉取镜像,imageName为镜像名，比如ubuntu,tag为标签，如果没有指定，默认为latest
docker pull imageName:tag

# 删除本地镜像,可指定镜像名和标签，也可以指定镜像Id
docker rmi [imageName:tag | imageId]

```
#### docker容器相关的命令

``` sh
#基于镜像创建容器并运行容器
# -it 表示终端交互式操作 （interactive terminal）
#  ubuntu:latest指定待运行的镜像
# /bin/bash 表示容器启动后要执行的命令
docker run -it ubuntu:latest /bin/bash

# 查看运行中的容器
docker ps

# 查看所有的容器
docker ps -a

# 停止容器
docker stop containterId

# 启动容器
docker start containerName

# 删除容器,删除容器前必须停止容器，否则报错
docker rm containerId

# 容器重命名(docker run之后，如果不指定--name参数，则会随机生成一个容器名)
docker rename oldName newName

# 进入容器后，通过第一次exit可以退出容器
```

### 安装和配置基于docker的Redis环境

``` sh
# 下载最新redis镜像
docker pull redis:latest

# 查看镜像
docker images

# 启动redis容器
# -it表示终端交互式操作，d表示后台运行
# --name 指定容器名称
# -p 指定容器的6379端口映射到宿主机（运行docker的机器）6379端口
# redis:latest 为启动容器的镜像
docker run -itd --name firstRedis -p 6379:6379 redis:latest


# 查看容器日志 
docker logs firstRedis

# 进入容器
# docker exec 表示在运行的容器中执行命令
# -it 表示以交互式终端执行命令
# firstRedis为容器名
# /bin/bash 为需要执行的命令
docker exec -it firstRedis /bin/bash

# 通过redis-cli命令进入redis客户端，可以执行set ,get命令

# 通过两次exit可以退出到windows命令行。第一次exit是退出redis-cli客户端到容器内部，第二次exit是退出容器。

# 停止容器，可以通过docker ps -a 对比容器前后状态
docker stop firstRedis

# 启动容器
docker start firstRedis

# 重启容器
docker restart firstRedis

## start和restart的区别是：start会挂载容器所关联的文件系统，restart不会。如果更改了redis启动时需要加载的配置项参数，那么重启时需要先stop，再start。

# 查看redis版本
docker start firstRedis
docker exec -it firstRedis /bin/bash

# 查看redis客户端版本
redis-cli --version

# 查看reis服务端版本
redis-server --version

# 如果要停止redis服务，可以通过exit退出容器，也可以通过redis-cli的shutdown命令关闭服务。
```

## 实践Redis的基本数据类型
基本数据类型包括字符串（string）类型、哈希(hash)类型、列表(list)类型、集合类型(set)和有序集合(sorted set或zset)类型

### 针对字符串的命令
#### 读写字符串的set和get命令
``` sh
 set key value [NX|XX] [GET] [EX seconds|PX milliseconds|EXAT unix-time-seconds|PXAT unix-time-milliseconds|KEEPTTL]

 # key和value分别是待设置字符串的键和值，value需要字符串类型。如果key对应的值已经存在，则再次set时会覆盖旧值。
 # nx (if not exist)： 如果不存在时再执行
 # xx ：如果存在则执行
 # get: 获取旧值，如果旧值不存在则返回nil
 # ex: 过期时间，单位为秒。必须是整数，不能是表达式。
 # px: 过期时间，单位为微秒。必须是整数，不能是表达式。
 # exat: 在某个unix秒的时间点过期
 # pxat: 在某个unix毫秒的时间点过期
 # keepttl: 保留key之前旧值的生存时间。

# set之后，可以使用get获取key对应的值。

127.0.0.1:6379> set key val ex 100
OK
127.0.0.1:6379> ttl key
(integer) 96
127.0.0.1:6379> set key val1 keepttl
OK
127.0.0.1:6379> get key
"val1"
127.0.0.1:6379> ttl key
(integer) 78
127.0.0.1:6379>
```

#### 设置和获取多个字符串的命令
``` sh
 mset key value [key value ...]
 mget key [key ...]

 # mset mget不能指定ex,px（ex,px不报错但不生效）,nx,xx等参数（报错），

 127.0.0.1:6379> mset key1 val1 key2 value2 k3 v3
OK
127.0.0.1:6379> mget key1 key2 k3
1) "val1"
2) "value2"
3) "v3"
 ```

 #### 对整数类型的值进行增量和减量操作
 ``` sh
 # 增量和减量操作，都需要key对应的值为整数，否则报错。

 incr key 
 # 对key对应的值+1，如果key不存在，则默认key对应的值为0

 # incrby key increment
 # 对key对应的值+increment

 decr key
 decrby key increment
 #减量操作

127.0.0.1:6379> incr key
(integer) 1
127.0.0.1:6379> incrby key 9
(integer) 10
127.0.0.1:6379> decr key
(integer) 9
127.0.0.1:6379> decrby key 2
(integer) 7
127.0.0.1:6379>
 ```

 #### getset命令获取旧值设置新值

 ``` sh
 getset key value
 # 如果key不存在，则返回nil，并设置新值；如果key存在，则返回旧值，并设置新值

127.0.0.1:6379> get key
"1"
127.0.0.1:6379> getset key 3
"1"
127.0.0.1:6379> get key
"3"
 ```

 #### 针对字符串的其他操作

 ``` sh
getrange key start end
# 获取key对应值的子字符串，start和end表示起始位置，start从0开始，end也可以从右向左，最右边的索引为-1

127.0.0.1:6379> set key 029-12345678
OK
127.0.0.1:6379> getrange key 4 11
"12345678"
127.0.0.1:6379> getrange key 4 -1
"12345678"

setrange key offset value
# 相当于字符串的替换操作，从offset位置开始，把值替换为value

127.0.0.1:6379> setrange key 4 87654321
(integer) 12
127.0.0.1:6379> get key
"029-87654321"

strlen key
# 统计字符串长度的命令

 append key value
 # 将value追加到原值的末尾
 127.0.0.1:6379> append key ***
(integer) 15
127.0.0.1:6379> get key
"029-87654321***"
 ```

 ### 针对哈希类型变量的命令
#### 设置并获取哈希值
``` sh
 hset key field value [field value ...]
 # field 和value可以理解为对象的属性和属性值,返回值为属性的个数

hget key field
# hget必须指定key和字段名称

 127.0.0.1:6379> hset hkey name farb sex male
(integer) 2
127.0.0.1:6379> hget hkey name
"farb"
127.0.0.1:6379>
```
#### hsetnx命令

``` sh
 hsetnx key field value
 # 当key不存在或者key和field对应value不存在时，设置value，后面只能跟一对key和value

 127.0.0.1:6379> hget hkey name
"farb"
127.0.0.1:6379> hsetnx hkey name jack
(integer) 0
127.0.0.1:6379> hsetnx hkey age 18
(integer) 1
127.0.0.1:6379>
127.0.0.1:6379> hsetnx hNotExistKey name Alice
(integer) 1
127.0.0.1:6379> hget hNotExistKey name
"Alice"
```
#### key的相关操作
``` sh
hkeys key
# 查看哈希类型的所有字段field，找不到的话就返回empty array信息

127.0.0.1:6379> hkeys hkey
1) "name"
2) "sex"
3) "age"

hvals key
# 查看所有field对应的值，找不到的话就返回empty array信息
127.0.0.1:6379> hvals hkey
1) "farb"
2) "male"
3) "18"

hgetall key
# 以键值对的形式查看key对应的哈希类型数据，找不到的话就返回empty array信息

127.0.0.1:6379> hgetall hkey
1) "name"
2) "farb"
3) "sex"
4) "male"
5) "age"
6) "18"
```
#### hexists命令判断值是否存在
``` sh
hexists key field
# 判断key和field对应的value是否存在,key不存在也返回0

127.0.0.1:6379> hexists hkey name
(integer) 1
127.0.0.1:6379> hexists hkey height
(integer) 0
127.0.0.1:6379> hexists nonExists name
(integer) 0

```

#### 哈希类型数据的删除操作
``` sh
hdel key field [field ...]
# 删除key指定的field数据，可以传入多个字段,返回值删除的字段数

127.0.0.1:6379> hdel hkey sex age
(integer) 2
127.0.0.1:6379> hgetall hkey
1) "name"
2) "farb"

```

### 针对列表类型变量的命令
#### 读写列表的命令
``` sh
 lpush key element [element ...]
 # 将元素依次插入到名为key的列表左侧

 lindex key index
 # 从键名为key的列表中从左侧读取第index个元素，下标从0开始

 127.0.0.1:6379> lpush mylist 1 2 3 4
(integer) 4
127.0.0.1:6379> lrange mylist 0 -1
1) "4"
2) "3"
3) "2"
4) "1"
127.0.0.1:6379> lindex mylist 1
"3"

 rpush key element [element ...]
 # 将元素依次插入到名为key的列表右侧

127.0.0.1:6379> rpush mylist 2 3 4
(integer) 7
127.0.0.1:6379> lrange mylist 0 -1
1) "4"
2) "3"
3) "2"
4) "1"
5) "2"
6) "3"
7) "4"
127.0.0.1:6379>
```

#### lpushx 和rpushx
``` sh
 lpushx key element [element ...]
 # 当key存在时，在key对应的列表左侧添加元素

127.0.0.1:6379> lpushx mylist 5 6
(integer) 9
127.0.0.1:6379> lrange mylist 0 -1
1) "6"
2) "5"
3) "4"
4) "3"
5) "2"
6) "1"
7) "2"
8) "3"
9) "4"

 rpushx key element [element ...]
 # 当key存在时，在key对应的列表右侧添加元素

127.0.0.1:6379> rpushx mylist 5 6
(integer) 11
127.0.0.1:6379> lrange mylist 0 -1
 1) "6"
 2) "5"
 3) "4"
 4) "3"
 5) "2"
 6) "1"
 7) "2"
 8) "3"
 9) "4"
10) "5"
11) "6"
```

#### 用list模拟堆栈和队列
``` sh
# 栈：后进先出，所以可以使用lpush、lpop命令或者rpush、rpop命令模拟栈
127.0.0.1:6379> lpush stack1 1 2
(integer) 2
127.0.0.1:6379> lrange stack1 0 -1
1) "2"
2) "1"
127.0.0.1:6379> lpop stack1
"2"

# 队列：先进先出，所以可以使用lpush、rpop或者rpush、lpop命令来模拟队列
127.0.0.1:6379> rpush queue 1 2
(integer) 2
127.0.0.1:6379> lpop queue
"1"
127.0.0.1:6379>
```

#### 用lrange获取指定区间内的数据
``` sh
 lrange key start stop
 # key为list键名，start为起始位置，从0开始，stop为结束位置，最后一个元素可以为-1

 127.0.0.1:6379> lrange mylist 0 -1
 1) "6"
 2) "5"
 3) "4"
 4) "3"
 5) "2"
 6) "1"
 7) "2"
 8) "3"
 9) "4"
10) "5"
11) "6"
127.0.0.1:6379> lrange mylist 1 3
1) "5"
2) "4"
3) "3"
127.0.0.1:6379>
```

#### 用lset修改列表数据
``` sh
 lset key index element
 # 将名为key的列表的第index个元素修改为element

 127.0.0.1:6379> lset mylist 0 7
OK
127.0.0.1:6379> lset mylist -1 7
OK
127.0.0.1:6379> lrange mylist 0 -1
 1) "7"
 2) "5"
 3) "4"
 4) "3"
 5) "2"
 6) "1"
 7) "2"
 8) "3"
 9) "4"
10) "5"
11) "7"
```

#### 删除列表数据
``` sh
 rpop key [count]
 # 从右边弹出count个元素

 lpop key [count]
 # 从左边弹出count个元素

 lrem key count element
 # 当count=0时，删除该列表中所有值是element的元素；
 # 当count>0时，从左到右删除数量为count个、值是element的元素；
 # 当count<0时，从右往左删除数量为count个、值是element的元素；

 127.0.0.1:6379> lpop mylist
"7"
127.0.0.1:6379> rpop mylist
"7"
127.0.0.1:6379> lrem mylist 0 5
(integer) 2
127.0.0.1:6379> lrange mylist 0 -1
1) "4"
2) "3"
3) "2"
4) "1"
5) "2"
6) "3"
7) "4"
```

### 针对集合的命令
#### 读写集合
``` sh
 sadd key member [member ...]
# 给名为key的集合中添加元素，元素会自动去重
 smembers key
 # 读取集合key中的所有元素

127.0.0.1:6379> sadd myset 1 2 2 3
(integer) 3
127.0.0.1:6379> smembers myset
1) "1"
2) "2"
3) "3"
```

#### 列表和集合类数据的区别
1. 列表存储数据时具有有序性，要么从左侧push，要么从右侧push。而集合不具有有序性。
2. 列表存储的数据可以存在重复元素，而集合会自动去重。

#### 用sismember判断元素是否存在
``` sh
sismember key member
# 判断集合key中是否存在member,存在返回1，不存在返回0

127.0.0.1:6379> sismember myset 2
(integer) 1
127.0.0.1:6379> sismember myset 4
(integer) 0

```

#### 获取集合的交集、并集和差集
``` sh
sinter key [key ...]
# 获取多个key对应集合的交集

127.0.0.1:6379> sadd myset2 2 3 4
(integer) 3
127.0.0.1:6379> sinter myset myset2
1) "2"
2) "3"

sunion key [key ...]
# 获取多个集合对应的并集
127.0.0.1:6379> sunion myset myset2
1) "1"
2) "2"
3) "3"
4) "4"

sdiff key [key ...]
# 获取多个key对应的差集
127.0.0.1:6379> sdiff myset myset2
1) "1"
127.0.0.1:6379> sdiff myset2 myset
1) "4"
```

#### 用srem命令删除集合数据
``` sh
srem key member [member ...]
# 删除集合key中的元素，并返回删除的元素个数

127.0.0.1:6379> srem myset 1
(integer) 1
```

### 有序集合的命令
#### 有序集合的读写
``` sh
 zadd key [NX|XX] [GT|LT] [CH] [INCR] score member [score member ...]
 # 给名为key的集合添加分数和元素
 # NX 有序集合中的元素不存在时才添加
 # XX 有序集合中的元素存在时添加
 # GT 当元素存在，且当分数大于旧分数时执行，不阻止添加元素。
 # LT 当元素存在，且当分数小于旧分数时执行，不阻止添加元素。
 # CH 修改返回的值为添加的元素数量+修改分数的元素数量（分数相同的时不更新）
 # INCR 给分数添加，使用此参数时，只能添加一个元素
 # score 分数或权重
 # member 元素

zrange key start stop [BYSCORE|BYLEX] [REV] [LIMIT offset count] [WITHSCORES]
# 读取名为key的有序集合中score排名区间在start到stop之间的数据
# ByScore 按分数范围查询
# Rev 是否反序
# Limit offset count 和mysql的分页查询一样
# withScores 返回分数

127.0.0.1:6379> zadd sortedSet 6 farb
(integer) 1
127.0.0.1:6379> zadd sortedSet GT 5 farb
(integer) 0
127.0.0.1:6379> zadd sortedSet GT 8 farb
(integer) 0
127.0.0.1:6379> zrange sortedSet -1 100
1) "farb"
127.0.0.1:6379> zrange sortedSet -1 100 WITHScores
1) "farb"
2) "8"
127.0.0.1:6379> zadd sortedSet GT 2 jack 3 lucy
(integer) 2
127.0.0.1:6379> zrange sortedSet 0 100 REV WITHSCORES
1) "farb"
2) "8"
3) "lucy"
4) "3"
5) "jack"
6) "2"
127.0.0.1:6379> zrange sortedSet 0 100 byscore
1) "jack"
2) "lucy"
3) "farb"
127.0.0.1:6379> zrange sortedSet 0 100 byscore withScores
1) "jack"
2) "2"
3) "lucy"
4) "3"
5) "farb"
6) "8"
```

#### zincrby修改元素的分数
```sh
zincrby key increment member
# 给名为key的有序集合中的member元素增加分数increment,返回值是最终分数

127.0.0.1:6379> zincrby sortedSet 10 farb
"18"
```

#### zscore 获取指定元素的分数
```sh
zscore key member
# 只能返回一个元素的分数，否则报错

127.0.0.1:6379> zscore sortedSet farb
"18"
```

#### zrank查看有序集合中的排名
```sh
 zrank key member [WITHSCORE]
# 正序排名，索引从0开始
zrevrank key member [WITHSCORE]
# 倒序排名

 127.0.0.1:6379> zrank sortedSet jack withScore
1) (integer) 0
2) "2"
127.0.0.1:6379> zrank sortedSet farb withScore
1) (integer) 2
2) "18"
127.0.0.1:6379> zrevrank sortedSet farb WithScore
1) (integer) 0
2) "18"
```

#### 删除有序集合中的值
```sh
 zrem key member [member ...]
 # 一次允许删除多个元素

 127.0.0.1:6379> zrem sortedSet jack lucy
(integer) 2
127.0.0.1:6379> zrange sortedSet 0 100
1) "farb"

 zremrangebyrank key start stop
 # 删除排名在start和stop之间的元素

127.0.0.1:6379> zadd myzset 1 one 2 two 3 three 4 four
(integer) 4
127.0.0.1:6379> zrange myzset 1 4
1) "two"
2) "three"
3) "four"
127.0.0.1:6379> zrange myzset 0 4
1) "one"
2) "two"
3) "three"
4) "four"
127.0.0.1:6379> zrange myzset 0 3
1) "one"
2) "two"
3) "three"
4) "four"
127.0.0.1:6379> zremrangebyrank myzset 0 2
(integer) 3
127.0.0.1:6379> zrange myzset 0 3
1) "four"

zremrangebyscore key min max
# 删除分数在min和max之间的元素

127.0.0.1:6379> zrange myzset 0 100 withScores
1) "one"
2) "1.5"
3) "two"
4) "2.6"
5) "three"
6) "3.7"
7) "four"
8) "4.8"
127.0.0.1:6379> zremrangebyscore myzset 2 4
(integer) 2
127.0.0.1:6379> zrange myzset 0 100 withScores
1) "one"
2) "1.5"
3) "four"
4) "4.8"
```