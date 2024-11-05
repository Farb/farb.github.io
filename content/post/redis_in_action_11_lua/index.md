---
title: 基于Docker的Redis实战-Redis整合Lua脚本
description:
slug: redis_in_action_11_lua
date: 2024-11-5
image: 
categories:
    - Redis
tags:
    - redis
    - docker
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---
源码链接： https://gitee.com/farb/architect-practicer-code 


## 在Redis中调用Lua脚本

### 引入Lua脚本的场景
 1. 重复执行相同类型的命令，比如缓存1到1000到内存中
 2. 在高并发场景下减少网络调用的开销，比如将多条redis命令放到一个Lua脚本中执行，只需要一次网络调用开销即可
 3. Redis会把lua脚本当作一个整体执行，天然具有原子性。

**注意：如果运行的lua脚本没有响应或不返回值，就会阻塞整个redis服务，并且运行lua脚本时很难调试，故lua脚本的代码要尽量简洁清晰**

### 通过redis-cli命令运行lua脚本

```lua
 //  先在本地路径D:\ArchitectPracticer\Redis\lua 创建一个SimpleRedis.lua文件，代码如下：
redis.call('set', 'name', 'farb')

// lua脚本里使用redis.call()方法调用redis命令，第一个参数是Redis命令，后面是命令的参数
```

```bash

# 启动一个redis容器，将本地磁盘的D:\ArchitectPracticer\Redis\lua路径挂载到容器的/luaScript目录下
PS D:\code\blogs\farb.github.io> docker run -itd --name redis-lua -v D:\ArchitectPracticer\Redis\lua:/luaScript:rw -p 6379:6379 redis:latest
c9900020aba5110bb43cf2ac6f1774f03a74940ff02690bdb3bf357222dfaff1

# 启动一个容器的交互式shell
PS D:\code\blogs\farb.github.io> docker exec -it redis-lua bash

# 使用--eval参数执行lua脚本，该脚本没有返回值，所以返回nil
root@c9900020aba5:/data# redis-cli --eval /luaScript/SimpleRedis.lua
(nil)

# 可以通过get命令查看是否写入成功
root@c9900020aba5:/data# redis-cli
127.0.0.1:6379> get name
"farb"

# 如果是Redis低版本，修改了lua脚本，需要重启容器，否则不会生效；我当前的Redis版本是7.2.5,不需要重启docker容器就能生效
PS D:\code\blogs\farb.github.io> docker restart redis-lua
redis-lua
PS D:\code\blogs\farb.github.io> docker exec -it redis-lua bash
root@c9900020aba5:/data# redis-cli --eval /luaScript/SimpleRedis.lua
(nil)
root@c9900020aba5:/data# redis-cli
127.0.0.1:6379> get name
"Jack"
```

### 直接通过eval 命令执行lua脚本
实际项目中，如果lua脚本里包含的语句较多，则通过lua脚本文件的方式执行会比较方便。如果语句很少，则可以直接通过eval命令执行lua脚本

```bash
# script是双引号括起来的脚本，numkeys是key的数量，这两个参数是必须的
eval script numkeys [key [key ...]] [arg [arg ...]]

root@c9900020aba5:/data# redis-cli
127.0.0.1:6379> eval "redis.call('set','age',18)" 0
(nil)
127.0.0.1:6379> get age
"18"
```

### 通过return返回脚本运行结果

新建一个ReturnRedisResult.lua文件，代码如下：`return 1`

```bash
# 执行lua脚本，返回1
root@c9900020aba5:/data# redis-cli --eval /luaScript/ReturnRedisResult.lua
(integer) 1

# 修改ReturnRedisResult.lua为return redis.call("set", "name", "HanMeimei")，直接返回OK
root@c9900020aba5:/data# redis-cli --eval /luaScript/ReturnRedisResult.lua
OK
```

### Redis中和lua相关的命令
1. 使用`Script Load <script>`事先装载lua脚本，然后使用evalsha命令多次运行该脚本。
2. 使用`Script Flush`清空Redis服务器的所有lua脚本。
3. 使用`Script Kill`终止正在运行的lua脚本。

```bash
# evalsha命令:sha1是lua脚本的sha1值，numkeys是key的数量
evalsha sha1 numkeys [key [key ...]] [arg [arg ...]]

PS D:\code\blogs\farb.github.io> docker exec -it redis-lua bash
root@c9900020aba5:/data# redis-cli

# 将lua脚本加载到redis服务器，返回值是sha1值
127.0.0.1:6379> script load "return 1"
"e0e1f9fabfc9d4800c877a703b823ac0578ff8db"

# 使用evalsha sha1值调用lua脚本，返回值是1，可重复调用
127.0.0.1:6379> evalsha e0e1f9fabfc9d4800c877a703b823ac0578ff8db 0
(integer) 1
127.0.0.1:6379> evalsha e0e1f9fabfc9d4800c877a703b823ac0578ff8db 0
(integer) 1

# 删除缓存中的所有lua脚本
127.0.0.1:6379> script flush
OK

# 再次执行lua脚本时报错
127.0.0.1:6379> evalsha e0e1f9fabfc9d4800c877a703b823ac0578ff8db 0
(error) NOSCRIPT No matching script. Please use EVAL.
```

### 观察Lua脚本阻塞redis的效果

```bash
# 执行一个死循环的lua脚本,由于redis的执行引擎是单线程的，因此整个Redis服务都会被阻塞，导致其他命令无法执行
127.0.0.1:6379> eval "while true do end" 0

# 使用ctrl + c 退出,重新连接一个客户端
^Croot@c9900020aba5:/data# redis-cli
127.0.0.1:6379> get name
# 提示Redis正在忙于执行一个脚本，只能使用Script Kill终止正在执行的脚本或者shutdown nosave来停止服务
(error) BUSY Redis is busy running a script. You can only call SCRIPT KILL or SHUTDOWN NOSAVE.
127.0.0.1:6379> script kill
OK

# 终止运行的脚本后，可以正常访问redis了。
127.0.0.1:6379> get name
"HanMeimei"

```

因此使用lua脚本需要格外小心，执行时间不能过长，否则不仅redis服务器阻塞，所有请求的线程也会阻塞，如果并发量巨大，会导致大量线程阻塞，最终系统瘫痪无法对外服务，这是很严重的事故。因此lua脚本要尽量保持短小，其次，逻辑不要太复杂，否则可能出现bug导致长时间运行，如果很复杂，需要充分测试。