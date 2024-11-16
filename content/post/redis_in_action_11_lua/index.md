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


## Redis整合lua高级实战

### 通过KEYS和ARGV传递参数

KEYS和ARGV通过都是数组传参，KEYS是传入redis命令需要的参数，ARGV是传入自定义的参数，但是下标是从1开始的。且eval命令传入的参数个数表示的KEYS的参数个数，不包括ARGV的参数个数。

```sh
root@c9900020aba5:/data# redis-cli
# 1表示key的数量，key1是key的名称，one是参数1，two是参数2
127.0.0.1:6379> eval "return {KEYS[1],ARGV[1],ARGV[2]}" 1 key1 one two
1) "key1"
2) "one"
3) "two"
# 2表示key的数量，key1是key的名称，读取第二个参数时发现是ARGV而不是KEY类型的，所以抛弃，two是参数2
127.0.0.1:6379> eval "return {KEYS[1],ARGV[1],ARGV[2]}" 2 key1 one two
1) "key1"
2) "two"
```

再看一个通过lua脚本传参的例子：

新建一个lua脚本KeysAndArgs.lua文件，代码如下：
```lua
redis.call('set','val',KEYS[1])
redis.call('set','argv',ARGV[1])
return ARGV[2];
```
在redis命令行中执行：

```sh
root@c9900020aba5:/data# redis-cli --eval  /luaScript/KeysAndArgv.lua 1 2 3
(error) ERR Lua redis lib command arguments must be strings or integers script: 3d8e39570f8dcd82458229647a72152f68fd74b7, on @user_script:2.

# 报错的原因是：脚本的第二行有错误，因为redis的命令传参只能使用KEYS，而不能用ARGV。

```

修正lua脚本再次执行：

```lua
redis.call('set','val',KEYS[1]);
redis.call('set','argv',KEYS[2]);
return {KEYS[1],ARGV[1]};
```

执行lua脚本：

```sh
# 这里有个坑人的地方，就是keys和argv的参数必须通过逗号隔开，而且每个传递的参数直接必须有空格，这一点格外注意
root@c9900020aba5:/data# redis-cli --eval /luaScript/KeysAndArgv.lua 1 2,3
1) "1"
root@c9900020aba5:/data# redis-cli --eval /luaScript/KeysAndArgv.lua 1 2, 3
1) "1"
root@c9900020aba5:/data# redis-cli --eval /luaScript/KeysAndArgv.lua 1 2 , 3
1) "1"
2) "3"
```


### 在脚本中引入分支语句

新建一个IfDemo.lua的脚本：

```lua
--  注意语法，if then else end关键字
if redis.call("exists","Name")==1 then
    return "Existed"
else
    return "No Name"
end
```

```bash
root@c9900020aba5:/data# redis-cli --eval /luaScript/IfDemo.lua 
"No Name"
root@c9900020aba5:/data# redis-cli
127.0.0.1:6379> set Name FARB
OK
127.0.0.1:6379> exit
root@c9900020aba5:/data# redis-cli --eval /luaScript/IfDemo.lua 
"Existed"
```

### 在脚本中引入循环语句
新建一个whileDemo.lua的脚本：

```lua
local times=0
while(time<100)
do
    -- 循环向redis中存1:1 到99：99的数据
    redis.call("set",times,times)
    times++
end
return 'Success'
```

执行lua脚本：

```bash
root@c9900020aba5:/data# redis-cli --eval /luaScript/WhileDemo.lua
"Success"
root@c9900020aba5:/data# redis-cli
127.0.0.1:6379> get 1
"1"
127.0.0.1:6379> get 99
"99"
```

### 在脚本中引入for循环语句
新建ForDemo.lua的脚本：

```lua
-- 初始值i=0,结束值为100（不包括），步长为1
for i=0,100,1 do
  redis.call('del',i)
end
return 'Success'
```

执行lua脚本：

```bash
root@c9900020aba5:/data# redis-cli --eval /luaScript/ForDemo.lua 
"Success"
root@c9900020aba5:/data# redis-cli
# 可以看到，上面的循环已经把1到99的数据删除了
127.0.0.1:6379> get 1
(nil)
127.0.0.1:6379> get 99
(nil)
```

### 在Java程序中调用Redis脚本

```java 
public class InvokeLua {
    /**
     * 主程序入口
     * 本程序演示了如何使用Jedis客户端与Redis服务器交互，执行Lua脚本
     *
     * @param args 命令行参数
     * @throws Exception 如果与Redis服务器的连接或执行脚本时发生错误
     */
    public static void main(String[] args) throws Exception {
        // 连接Redis服务器
        Jedis jedis = new Jedis("localhost");

        // 定义Lua脚本，该脚本将两个键分别设置为两个值，并返回第一个参数
        String script = "redis.call('set','k1',KEYS[1]);redis.call('set','k2',KEYS[2]);return ARGV[1];";

        // 执行Lua脚本，传递键值和参数
        // 这里使用eval命令执行脚本，keys参数指定脚本中使用的键，argv参数指定脚本中的参数
        String result = jedis.eval(script, List.of("v1", "v2"), List.of("arg1")).toString();

        // 输出执行结果 arg1
        System.out.println(result);
    }
}

```

可以看到redis中也已经生效：

```bash
127.0.0.1:6379> get k1
"v1"
127.0.0.1:6379> get k2
"v2"
```

### lua脚本有错，还会执行吗？

**经实验，redis.call()命令一旦发生错误，就会导致整个脚本的停止，已经执行的命令不会回滚。redis.pcall()命令执行报错时，会继续执行。**
```java
/**
 * LuaWithError 类用于演示在 Redis 中使用 Lua 脚本时的错误处理
 * 该类通过 Jedis 客户端与 Redis 进行交互，并尝试执行一个包含错误的 Lua 脚本
 */
public class LuaWithError {
    /**
     * 主函数执行一个包含错误的 Lua 脚本
     * 该脚本试图在 Redis 中设置三个键值对，其中一个操作故意省略了值参数以引发错误
     *
     * @param args 命令行参数，未使用
     */
    public static void main(String[] args) {
        // 创建 Jedis 实例，连接到本地 Redis 服务器
        Jedis jedis = new Jedis("localhost");
        // 定义一个包含错误的 Lua 脚本，故意在第二个 set 操作中省略了值参数
        String script = "redis.call('set','Name','Farb');redis.call('set','Error');redis.call('set','Next','Next');";
        // 执行 Lua 脚本并打印结果，预期会遇到错误
        System.out.println(jedis.eval(script));
    }
}

```

**报错信息如下：**

```log
Exception in thread "main" redis.clients.jedis.exceptions.JedisDataException: ERR Wrong number of args calling Redis command from script script: b9b88af25b4f44f74fda6dc5189e405306db5a5f, on @user_script:1.
	at redis.clients.jedis.Protocol.processError(Protocol.java:132)
	at redis.clients.jedis.Protocol.process(Protocol.java:166)
	at redis.clients.jedis.Protocol.read(Protocol.java:220)
	at redis.clients.jedis.Connection.readProtocolWithCheckingBroken(Connection.java:278)
	at redis.clients.jedis.Connection.getOne(Connection.java:256)
	at redis.clients.jedis.Jedis.getEvalResult(Jedis.java:2883)
	at redis.clients.jedis.Jedis.eval(Jedis.java:2861)
	at redis.clients.jedis.Jedis.eval(Jedis.java:2874)
	at chapter11.LuaWithError.main(LuaWithError.java:22)
```

```bash
# 可以看到，redis.call()命令执行时，已经执行了第一个set操作，但是第二个set操作执行时出错，导致整个脚本的停止。
127.0.0.1:6379> get Name
"Farb"
127.0.0.1:6379> get Next
(nil)
```

将上面代码中的redis.call改为redis.pcall()，可以避免脚本停止运行。程序没有抛出异常，打印返回nil

```java
    public static void main(String[] args) {
        // 创建 Jedis 实例，连接到本地 Redis 服务器
        Jedis jedis = new Jedis("localhost");
        // 定义一个包含错误的 Lua 脚本，故意在第二个 set 操作中省略了值参数
        String script = "redis.pcall('set','Name','Farb');redis.pcall('set','Error');redis.pcall('set','Next','Next');";
        // 执行 Lua 脚本并打印结果，预期会遇到错误
        System.out.println(jedis.eval(script, 0));
    }
```

```bash
# 可以看到，redis.pcall()命令执行时，已经执行了第一个set操作，但是第二个set操作执行时出错，但是并没有导致整个脚本的停止，第三个set操作执行成功。
127.0.0.1:6379> get Name
"Farb"
127.0.0.1:6379> Get Next
"Next"
```