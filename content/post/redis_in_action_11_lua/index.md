---
title: 基于Docker的Redis实战-Redis整合Lua脚本
description:
slug: redis_in_action_11_lua
date: 2024-11-05
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

执行bash脚本

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

## Redis整合lua脚本实战

### 以计数模式实现限流效果
限流是指某应用模块需要限制指定IP（或指定模块、指定应用）在单位时间内的访问次数。

lua脚本如下：

```lua
-- 从 KEYS 数组中获取模块名
local moduleName=KEYs[1]
-- 将限流次数从字符串转换为数字
local limitCount=tonumber(ARGV[1])
-- 获取当前模块的计数，如果不存在则默认为0
local current=tonumber(redis.call('get',moduleName) or "0")

-- 检查当前计数是否超过了限流次数
if(current+1>limitCount) then
    -- 如果超过，则返回0表示操作失败
	return 0
else
    -- 如果未超过，则增加模块的计数
    redis.call('incrby',moduleName,1)
    -- 更新模块的过期时间
    redis.call('expire',moduleName,tonumber(ARGV[2]))
	-- 返回增加后的计数
	return current+1
end

```

java代码如下：

```java
public class LimitByCount {
    /**
     * 判断是否可以访问
     * 通过Redis和Lua脚本确保在限定时间内访问次数不超过限定次数
     *
     * @param jedis      Redis客户端，用于执行Redis命令
     * @param moduleName 模块名称，用于在Redis中区分不同的模块
     * @param limitTime  限定时间，单位为秒，在这个时间内访问次数不能超过limitNum
     * @param limitNum   限定次数，在limitTime内允许的最大访问次数
     * @return 如果当前访问未超过限制，则返回true，否则返回false
     */
    public static boolean canVisit(Jedis jedis, String moduleName, int limitTime, int limitNum) {
        // Lua脚本，用于在Redis中执行逻辑判断和数据更新
        // 这段脚本的作用是检查和更新模块的访问次数，确保在限定时间内访问次数不超过限定次数
        String luaScript = "local moduleName=KEYs[1]\n" +
                "local limitCount=tonumber(ARGV[1])\n" +
                "local current=tonumber(redis.call('get',moduleName) or \"0\")\n" +
                "if(current+1>limitCount) then\n" +
                "  return 0\n" +
                "else\n" +
                "  redis.call('incrby',moduleName,1)\n" +
                "  redis.call('expire',moduleName,tonumber(ARGV[2]))\n" +
                "  return current+1\n" +
                "end";
        // 执行Lua脚本，判断返回值,如果不为0，则表示可以访问
        String result = jedis.eval(luaScript, 1, moduleName, String.valueOf(limitNum), String.valueOf(limitTime)).toString();
        return !result.equals("0");
    }
}

/**
 * LuaLimitByCountThread 类继承自 Thread 类，用于演示如何通过 Redis 限流
 * 它在每个线程中检查是否可以访问某个资源，基于计数的限流策略
 */
public class LuaLimitByCountThread extends Thread {
    /**
     * 线程的入口点
     * 它创建一个 Jedis 实例来与 Redis 通信，并循环检查当前线程是否可以访问资源
     * @see Thread#run()
     */
    @Override
    public void run() {
        // 创建 Jedis 实例，连接到本地 Redis 服务器
        Jedis jedis = new Jedis("localhost");
        // 循环 5 次，检查是否可以访问资源
        for (int i = 0; i < 5; i++) {
            // 调用 LimitByCount 的 canVisit 方法，判断当前线程是否可以访问资源
            boolean canVisit = LimitByCount.canVisit(jedis, Thread.currentThread().getName(), 10, 3);
            // 根据 canVisit 的结果，打印是否可以访问
            if (canVisit) {
                System.out.println(Thread.currentThread().getName() + " can visit");
            } else {
                System.out.println(Thread.currentThread().getName() + " can not visit");
            }
        }
    }

    /**
     * 程序的入口点
     * 它创建并启动多个 LuaLimitByCountThread 线程，以演示多线程环境下的限流效果
     * @param args 命令行参数
     */
    public static void main(String[] args) {
        // 循环 3 次，创建并启动 3 个线程
        for (int i = 0; i < 3; i++) {
            new LuaLimitByCountThread().start();
        }
    }
}

```

如预期，每个模块只允许10秒内被访问3次，超过次数的线程会被拒绝访问。运行结果如下：

```log
Thread-1 can visit
Thread-2 can visit
Thread-0 can visit
Thread-0 can visit
Thread-2 can visit
Thread-1 can visit
Thread-0 can visit
Thread-2 can visit
Thread-1 can visit
Thread-0 can not visit
Thread-2 can not visit
Thread-1 can not visit
Thread-2 can not visit
Thread-1 can not visit
Thread-0 can not visit
```

### lua脚本防止超卖
超卖是指在某一时间段内，销售数量超过库存数量。

lua脚本具有天然原子性，redis的读写是单线程的，所以使用lua脚本可以防止超卖。

lua脚本如下：

```lua
-- 从 Redis 中获取指定键的当前值，并将其转换为数字
local left = tonumber(redis.call('get',KEYS[1]) or '0')

-- 检查当前值是否小于等于0
if(left<=0) then
    -- 如果值小于等于0，返回0
    return 0
else
    -- 如果值大于0，将键的值减少1，并返回减少前的值
    redis.call('decrby',KEYS[1],1)
    return left
end
```

java代码如下：

```java
/**
 * OverSellCheck 类用于检查是否可以购买商品，以防止过量销售
 */
public class OverSellCheck {

 /**
     * 检查用户是否可以购买商品
     *
     * @param jedis     Redis 客户端实例，用于与 Redis 数据库交互
     * @param goodsKey 商品键值
     * @return boolean 表示是否可以购买商品，true 表示可以购买，false 表示不可以购买
     */
    public static boolean canBuy(Jedis jedis, String goodsKey) {
        // Lua 脚本用于原子操作，以防止并发条件下的超卖现象
        String lua = "local left = tonumber(redis.call('get',KEYS[1]))\n" +
                "if(left<=0) then\n" +
                "\treturn 0\n" +
                "else\n" +
                "  redis.call('decrby',KEYS[1],1)\n" +
                "\treturn left\n" +
                "end";
        // 执行 Lua 脚本，判断商品是否还可以购买
        String result = jedis.eval(lua, 1, goodsKey).toString();
        // 如果返回结果不为 "0"，则表示还可以购买商品
        return !result.equals("0");
    }
}

/**
 * OverSellDemo 类用于演示在多线程环境下可能出现的超卖问题
 * 它通过多个线程模拟高并发场景下的商品购买行为
 */
public class OverSellDemo extends Thread {
    // 商品的键，用于在Redis中标识商品
    private final static String GOODS_KEY = "goods:1";

    /**
     * 程序的入口点
     * 初始化商品库存，并启动多个线程模拟购买行为
     * @param args 命令行参数
     */
    public static void main(String[] args) {
        // 连接本地Redis服务器
        Jedis jedis = new Jedis("localhost");
        // 设置商品库存为5，过期时间为10秒
        jedis.setex(GOODS_KEY, 10, "5");
        // 启动10个线程模拟购买行为
        for (int i = 0; i < 10; i++) {
            new OverSellDemo().start();
        }
    }

    /**
     * 线程的运行方法
     * 每个线程尝试购买商品，并打印购买结果
     */
    @Override
    public void run() {
        // 连接本地Redis服务器
        Jedis jedis = new Jedis("localhost");
        // 检查当前是否可以购买商品
        boolean canBuy = OverSellCheck.canBuy(jedis, GOODS_KEY);
        if (canBuy) {
            // 如果可以购买，打印线程名称和购买结果
            System.out.println(Thread.currentThread().getName() + " can buy");
        } else {
            // 如果不可以购买，打印线程名称和购买结果
            System.out.println(Thread.currentThread().getName() + " can not buy");
        }
    }
}

```

可以看到，在多线程环境下，虽然有10个线程尝试购买商品，但只有5个线程可以购买成功，其余线程因为库存不足而被拒绝购买。 运行结果如下：

```log
Thread-2 can not buy
Thread-7 can buy
Thread-1 can buy
Thread-9 can buy
Thread-5 can not buy
Thread-0 can not buy
Thread-8 can not buy
Thread-3 can buy
Thread-6 can not buy
Thread-4 can buy
```


