---
title: 基于Docker的Redis实战--实践Redis的常用命令
description:
slug: redis_in_action_03_common_used_command
date: 2024-07-20
image: 
categories:
    - Redis
tags:
    - redis
    - docker
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---


### 键操作命令
#### exists判断键是否存在
```sh
exists key [key ...]
# 判断key是否存在，可以一次指定多个key。返回值是key存在的个数。

127.0.0.1:6379> exists key key1 key2 key3
(integer) 3
# 因为key3不存在，所以返回前3个存在的key数量。

```

#### keys命令查找键

```sh
 keys pattern
 # 可以使用通配符或正则表达式查找键，？代替一位字符，*匹配零个、一个或多个字符。
 # 注意：keys * 可以返回所有的键，但是项目中的键可能很多，全部返回没有意义，生产环境还会造成性能风险。
127.0.0.1:6379> keys key?
1) "key1"
2) "key2"
127.0.0.1:6379> keys key*
1) "key1"
2) "key"
3) "key2"
127.0.0.1:6379> keys key
1) "key"
127.0.0.1:6379> keys *
1) "blogUrl"
2) "hkey"
3) "key1"
4) "k3"
5) "key"
6) "key2"
7) "sortedSet"
```

#### scan命令查找键
```sh
scan cursor [MATCH pattern] [COUNT count] [TYPE type]
# scan类似于分页，cursor是游标的位置，从0开始启动迭代，
# 默认pattern返回所有key, 默认count每次返回10个key
# 返回的值是下次游标的迭代位置和key的数组，如果游标返回0，表明迭代结束
# keys是以阻塞的方式返回键，scan是以非阻塞的方式查找并返回键
# 如果待查找的键的数量很多耗时可能较长，Redis的单线程可能导致无法执行其他命令，
# 严重的话会导致系统卡顿，因此使用时注意是开发还是生产环境，键的数量多少。
127.0.0.1:6379> scan 0
1) "0"
2) 1) "key1"
   2) "hkey"
   3) "key"
   4) "key2"
   5) "blogUrl"
   6) "k3"
   7) "sortedSet"
127.0.0.1:6379> set name farb
OK
127.0.0.1:6379> set sex man
OK
127.0.0.1:6379> set age 18
OK
127.0.0.1:6379> set height 180
OK
127.0.0.1:6379> scan 0
1) "7"
2)  1) "key1"
    2) "hkey"
    3) "name"
    4) "key2"
    5) "key"
    6) "height"
    7) "blogUrl"
    8) "k3"
    9) "sex"
   10) "age"
127.0.0.1:6379> scan 7
1) "0"
2) 1) "sortedSet"
```

#### 重命名键
```sh
 rename key newkey
 # 将旧键key重命名为新键newKey，如果newKey之气已经存在，则会覆盖之前的值
 renamenx key newkey
 # 如果新键名newKey不存在则重命名

127.0.0.1:6379> rename sex gender
OK
127.0.0.1:6379> get gender
"man"
127.0.0.1:6379> get sex
(nil)

# name已存在，所以不会成功
127.0.0.1:6379> renamenx gender name
(integer) 0
127.0.0.1:6379> renamenx gender sex
(integer) 1
```

#### del命令删除键
```sh
del key [key ...]
# 支持一次性删除多个键，返回值删除键的个数

# 因为keyn不存在，key存在，故返回1
127.0.0.1:6379> del key keyn
(integer) 1
```

#### 关于键生存时间的命令
```sh
pttl key
ttl key
# pttl以毫秒返回，ttl以秒返回。
# key不存在，返回-2
# key存在但没设置生存时间，返回-1
# 可以使用set命令时通过ex或px指定key的生存时间
# 也可以通过 expire key seconds [NX|XX|GT|LT] 
# 或 pexpire key milliseconds [NX|XX|GT|LT]指定生存时间

127.0.0.1:6379> pttl key
(integer) -2
127.0.0.1:6379> pttl key1
(integer) -1
127.0.0.1:6379> set mykey myval ex 300
OK
127.0.0.1:6379> ttl mykey
(integer) 293
```
### HyperLogLog相关命令
HyperLogLog对象能高效地统计基数，每个HyperLogLog对象大概只需要12KB内存就能计算2^64的元素的基数。

#### pfadd添加键值对
```sh
pfadd key [element [element ...]]
# 可以对同一个键添加多个值

127.0.0.1:6379> pfadd Perter Math Computer Music
(integer) 1
127.0.0.1:6379> pfcount Perter
(integer) 3
```
#### pfcount统计基数值
```sh
 pfcount key [key ...]
 # 查看一个或多个键的基数,统计的是多个键中有多少个不重复的数据，是一个近似值，当基数量很大时结果未必精确。

 127.0.0.1:6379> pfadd set1 1 2 3
(integer) 1
127.0.0.1:6379> pfadd set2 2 4 5
(integer) 1
127.0.0.1:6379> pfcount set1 set2
(integer) 5
127.0.0.1:6379> pfcount nonExistKey
(integer) 0

```
#### pfmerge进行合并操作
```sh
pfmerge destkey [sourcekey [sourcekey ...]]
# 将一个或多个hyperLogLog对象合并成一个，destKey是合并后的HyperLogLog对象的键，
# 如果destKey不存在，则创建

127.0.0.1:6379> pfmerge set3 set1 set2
OK
127.0.0.1:6379> pfcount set3
(integer) 5
```

#### 统计网站访问总人数
```sh
127.0.0.1:6379> pfadd totalUserCount u1 u2 u3 u1 u3
(integer) 1
127.0.0.1:6379> pfcount totalUserCount
(integer) 3
# pfcount会自动去重同一个用户
```
### Lua脚本相关命令
lua是一种轻量的脚本语言，可以嵌入到应用程序中，能以较小的代价定制功能。

#### 把lua脚本装载到缓存里
```sh
script load scriptContent
# 把脚本scriptContent装载到缓存里，不执行脚本，返回脚本的sha1校验和

127.0.0.1:6379> script load 'return 1+2'
"e13c398af9f2658ef7050acf3b266f87cfc2f6ab"

# 判断指定校验和的脚本是否存在
127.0.0.1:6379> script exists e13c398af9f2658ef7050acf3b266f87cfc2f6ab
1) (integer) 1
```

#### evalsha执行缓存中的脚本
```sh
evalsha sha1 numkeys [key [key ...]] [arg [arg ...]]
#sha1为载入缓存中的lua脚本的校验和，numKeys是参数的个数
# key指定脚本中用到的键，arg指定脚本中的参数

127.0.0.1:6379> evalsha e13c398af9f2658ef7050acf3b266f87cfc2f6ab 0
(integer) 3
```
#### 清空缓存中的脚本
```sh
script flush [ASYNC|SYNC] 
# 清空缓存中的所有脚本，可以指定异步还是同步

127.0.0.1:6379> script exists e13c398af9f2658ef7050acf3b266f87cfc2f6ab
1) (integer) 1
127.0.0.1:6379> script flush
OK
127.0.0.1:6379> script exists e13c398af9f2658ef7050acf3b266f87cfc2f6ab
1) (integer) 0
```

#### eval 执行lua脚本
```sh
 eval script numkeys [key [key ...]] [arg [arg ...]]
 # script为脚本内容，numKeys为参数的个数
 # key为脚本中用的键，arg为脚本中的实参

127.0.0.1:6379> eval 'return {KEYS[1],ARGV[1]}' 1 name farb
1) "name"
2) "farb"

# KEYS 和ARGV是全局变量，是固定写法，区分大小写
127.0.0.1:6379> eval 'return KEYS[1],argv[1]' 1 age 18
(error) ERR user_script:1: Script attempted to access nonexistent global variable 'argv' script: 33f078271bc288ca9d4c04eab1a00d97877ae05c, on @user_script:1.
127.0.0.1:6379> eval 'return KEYS[1],ARGV[1]' 1 age 18
"age"

# 如果脚本出现死循环等需要结束脚本的运行，可以执行script kill
127.0.0.1:6379> script kill
(error) NOTBUSY No scripts in execution right now.
```

### 排序相关命令
可以对列表、集合和有序集合等数据进行升降序排列。

#### 用sort命令排序
```sh
sort key [BY pattern] [LIMIT offset count] [GET pattern [GET pattern ...]] [ASC|DESC] [ALPHA] [STORE destination]
# 可以通过asc参数升序排列，通过desc降序排列
127.0.0.1:6379> rpush salary 100 90 200 400 300
(integer) 5
# 原始顺序
127.0.0.1:6379> lrange salary 0 -1
1) "100"
2) "90"
3) "200"
4) "400"
5) "300"
# 排序后
127.0.0.1:6379> sort salary desc
1) "400"
2) "300"
3) "200"
4) "100"
5) "90" 

#对集合进行排序
127.0.0.1:6379> sadd nameSet Peter Tom Mary
(integer) 3
127.0.0.1:6379> sort nameSet asc
(error) ERR One or more scores can't be converted into double
# 默认是按照数值进行排序的，如果元素是字符串则报错，
# 需要使用alpha参数，表示按照字典序排序
127.0.0.1:6379> sort nameSet asc alpha
1) "Mary"
2) "Peter"
3) "Tom"

# 有序集合排序同理，即使有分数权重
127.0.0.1:6379> zadd nameZset 1 farb 2 jack 3 ben
(integer) 3
127.0.0.1:6379> sort nameZset asc
(error) ERR One or more scores can't be converted into double
127.0.0.1:6379> sort nameZset asc alpha
1) "ben"
2) "farb"
3) "jack"
```

#### 用by参数指定排序模式

```sh
# by后面可以匹配模式，包括正则表达式或通配符
127.0.0.1:6379> rpush vipLevel vip1 vip3 vip2
(integer) 3
127.0.0.1:6379> lrange vipLevel 0 -1
1) "vip1"
2) "vip3"
3) "vip2"
127.0.0.1:6379> sort vipLevel by vip*
1) "vip1"
2) "vip2"
3) "vip3"
```

#### 用limit参数返回部分排序结果
```sh
# [LIMIT offset count]
# 和mysql中的分页原理一致，offset表示跳过的元素个数，count表示要返回的个数
127.0.0.1:6379> rpush nums 1 9 0 4 8  3 2  7
(integer) 8
127.0.0.1:6379> sort nums limit 0 3 asc
1) "0"
2) "1"
3) "2"
```

#### sort命令里get参数的用法
```sh
127.0.0.1:6379> set 2 jack
OK
127.0.0.1:6379> set 1 farb
OK
127.0.0.1:6379> set 3 ben
OK
127.0.0.1:6379> rpush scores 2 3 1
(integer) 3

# sort后面加上get，会把排序结果的值作为键，用每个键再去获取值
# get后面使用*表示所有键，也可以使用其他匹配模式
127.0.0.1:6379> sort scores get *
1) "farb"
2) "jack"
3) "ben"
```

#### 通过store参数提升性能
```sh
# 通过对数据排序后，将结果缓存到其他key中，
# 这样可以避免对相同数据重复排序，从而提升性能
127.0.0.1:6379> sort scores desc store score-desc
(integer) 3
127.0.0.1:6379> lrange score-desc 0 -1
1) "3"
2) "2"
3) "1"
```