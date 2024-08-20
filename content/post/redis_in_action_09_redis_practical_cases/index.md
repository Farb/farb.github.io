---
title: 基于Docker的Redis实战-Redis综合实践案例
description:
slug: redis_in_action_09_redis_practical_cases
date: 2024-08-18
image: 
categories:
    - Redis
tags:
    - redis
    - docker
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---
源码链接： https://gitee.com/farb/architect-practicer-code 

### 一、Redis消息队列实战
> 实际项目中，模块间可以通过消息队列（Message queue,MQ）进行通信，MQ有各种实现，如RabbitMQ、Kafka等。使用场景比如：订单模块在处理好一个订单后可以把该订单对象放入消息队列，而记账模块则可以从消息队列中获取订单对象，进行记账。

#### 1.消息队列与Redis消息订阅发布模式
用消息队列来交互数据的好处是“解耦合”，比如订单模块和记账模块在交互数据时无需考虑对方的业务细节，一般会用现成的消息队列，如RabbitMQ、Kafka等。而Redis消息订阅发布模式也能实现消息队列的效果。

> Redis的消息订阅和发布模式是一种消息的通信模式，其中发布者（Publisher）可以向指定的频道（channel）发布消息，而订阅者（Subscriber）则可以订阅指定的频道，当有消息发布到该频道时，订阅者就会收到该消息。每个频道包含一个消息队列，当发布者发布消息时，消息会被推送到该频道的消息队列中，订阅者会从该队列中获取消息。

![](https://s3.bmp.ovh/imgs/2024/08/18/59bcc19d1652ae25.png)

#### 2.消息订阅发布的命令和流程

![](https://s3.bmp.ovh/imgs/2024/08/18/3c222d8834ae8cb6.png)

```sh
# 启动一个redis服务端容器
PS D:\code\blogs\farb.github.io> docker run -itd --name redisPublisher -p 6379:6379 redis
563f5a099d06e078e77d4ae1ebefef01def94316203ed1db750a73c4afb5e4de

# 获取redis服务端容器的IP
PS D:\code\blogs\farb.github.io> docker inspect -f "{{.NetworkSettings.IPAddress}}" redisPublisher
172.17.0.2

# 进入redis服务端容器，执行redis-cli命令
PS D:\code\blogs\farb.github.io> docker exec -it redisPublisher bash
root@563f5a099d06:/data# redis-cli

# 发布消息，channel是频道名称，message是消息内容。当前返回0，因为没有订阅者
127.0.0.1:6379> publish channel message
(integer) 0

# 在另一个命令行窗口运行另一个容器redisSub1，使用redis-cli连接到redisPublisher容器所在的服务器，订阅MQChannel频道，当前返回1，表示订阅成功
PS D:\code\blogs\farb.github.io> docker run -itd --name redisSub1 -p 6380:6380 redis
bc7c2748c6a82e1b1c8c06cde89865ea6963c8dba8c16016e65c35be3d8c16bd
PS D:\code\blogs\farb.github.io> docker exec -it redisSub1 bash
root@bc7c2748c6a8:/data# redis-cli -h 172.17.0.2 -p 6379
172.17.0.2:6379> subscribe MQChannel
1) "subscribe"
2) "MQChannel"
3) (integer) 1
Reading messages... (press Ctrl-C to quit or any key to type command)

# 回到redisPublisher容器，发布消息（注意，多个单词应该用引号），当前返回1，表示消息发送给1个订阅者
127.0.0.1:6379> publish MQChannel "Hello,MQ subscribers!"
(integer) 1 

# 在redisSub1容器中，可以看到接收到以下消息，1)message为消息类型，2)MQChannel为频道名称，3)Hello,MQ subscribers!为消息内容
1) "message"
2) "MQChannel"
3) "Hello,MQ subscribers!"

# 在另一个命令行窗口再启动另一个redisSub2容器，订阅MQChannel频道，当前返回1，表示订阅成功
PS D:\code\blogs\farb.github.io> docker run -itd --name redisSub2 redis
8a70f31c6f2d461e95fafbe2f79c499d13ff3fe4a12415fdb67a78e8efd7b899
PS D:\code\blogs\farb.github.io> docker exec -it redisSub2 bash
root@8a70f31c6f2d:/data# redis-cli -h 172.17.0.2 -p 6379
172.17.0.2:6379> subscribe MQChannel
1) "subscribe"
2) "MQChannel"
3) (integer) 1
Reading messages... (press Ctrl-C to quit or any key to type command)

# 再次回到redisPublisher容器，发布消息，当前返回2，表示消息发送给2个订阅者
127.0.0.1:6379> publish MQChannel "This is the 2nd message."
(integer) 2

# 回到reidsSub1和redisSub2容器中，可以看到接收到以下消息，1)message为消息类型，2)MQChannel为频道名称，3)This is the 2nd message.为消息内容
1) "message" 
2) "MQChannel"
3) "This is the 2nd message."
Reading messages... (press Ctrl-C to quit or any key to type command)
```

#### 3.消息订阅发布的相关命令汇总
```sh
# 发布消息，channel是频道名称，message是消息内容，多个单词应该用引号
publish channel message

# 订阅频道，channel是频道名称，可以同时指定多个频道
subscribe channel [channel ...]

# 取消订阅频道，channel是频道名称，可以同时指定多个频道
unsubscribe [channel [channel ...]]

# 使用模式的方式订阅频道，pattern是模式名称，可以同时指定多个模式。?表示匹配一个字符，*表示匹配任意多个字符
psubscribe pattern [pattern ...]

# 取消订阅模式，pattern是模式名称，可以同时指定多个模式
punsubscribe [pattern [pattern ...]]
```

#### 4.Java与消息队列的实战范例
确保pom.xml中包含以下依赖。

```xml
        <dependency>
            <groupId>redis.clients</groupId>
            <artifactId>jedis</artifactId>
            <version>3.3.0</version>
        </dependency>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson</artifactId>
            <version>1.2.83</version>
        </dependency>
```

创建一个Publisher类，用于发布消息。

```java
public class Publisher {
    /**
     * 主函数入口
     * @param args 命令行参数，本程序未使用
     */
    public static void main(String[] args) {
        // 创建一个HashMap对象用于存储订单信息
        HashMap<String,String> order = new HashMap<>();
        // 添加订单ID
        order.put("id","1");
        // 添加订单所有者
        order.put("owner","Farb");
        // 添加订单金额
        order.put("amount","10000");

        // 将订单信息转换为JSON字符串格式
        String jsonString = JSON.toJSONString(order);

        // 创建Jedis对象，连接本地Redis服务器
        Jedis jedis = new Jedis("127.0.0.1", 6379);
        // 发布JSON字符串到指定的Redis频道上
        jedis.publish("MQChannel", jsonString);
    }
}
```

创建一个Subscriber类，该类继承JedisPubSub抽象类，用于订阅消息、接收消息、取消订阅。
```java
public class Subscriber extends JedisPubSub {
    /**
     * 主函数入口
     *
     * @param args 命令行参数
     */
    public static void main(String[] args) {
        // 创建一个订阅者实例
        Subscriber subscriber = new Subscriber();
        // 连接到本地的Redis服务器
        Jedis jedis = new Jedis("localhost", 6379);
        // 订阅名为"MQChannel"的频道
        jedis.subscribe(subscriber, "MQChannel");
    }

    @Override
    public void onSubscribe(String channel, int subscribedChannels) {
        System.out.println("subscribe the channel" + channel);
    }

    @Override
    public void onUnsubscribe(String channel, int subscribedChannels) {
        System.out.println("unsubscribe the channel" + channel);
    }

    /**
     * 接收并处理来自指定频道的消息
     * 该方法通过解析接收到的消息内容，将其转换为HashMap对象，以便后续处理
     * 主要用于演示如何从消息中提取订单信息并进行相关操作
     *
     * @param channel 消息所属的频道，用于标识消息的来源
     * @param message 原始的字符串消息内容，通常为JSON格式字符串，包含订单信息
     */
    @Override
    public void onMessage(String channel, String message) {
        // 将消息内容解析为HashMap对象，以便于访问其中的特定字段
        HashMap<String, String> orderMap = JSON.parseObject(message, HashMap.class);

        // 从订单Map中获取并打印订单的ID、所有者和账户信息
        System.out.println("id=" + orderMap.get("id"));
        System.out.println("owner=" + orderMap.get("owner"));
        System.out.println("amount=" + orderMap.get("amount"));
    }
}
```

### 二、Java实战Redis分布式锁
> 在一台主机的多线程场景里，为了保护某个对象在同一时刻只能被一个线程访问，可以使用锁机制，即线程只有在获取该对象锁资源的前提下才能访问，在访问完以后需要立刻释放锁，以便其他线程继续使用该对象。
> 扩展到多台主机，如果访问同一对象的线程来自分布式系统的多台主机，那么用来确保访问唯一性的锁就叫分布式锁。

#### 1.观察分布式锁的特性
分布式锁工作在分布式系统里的高并发场景里，除了具有“加锁”和“解锁”的功能外，还需具备如下两个特性：

1. 限时等待。哪怕加锁的主机系统崩溃导致无法再发出"解锁"指令，加载在对象上的分布式锁也能在一定时间后自动解锁；
2. 需要确保加锁和解锁的主机必须唯一。比如主机1在发出“锁余额数据”指令的同时，发出“10秒后解锁”的指令，但是10秒后主机1并没有执行完操作余额数据的指令，此时主机1自动解锁，并且主机2得到了锁。当主机1执行完毕时，释放锁时，释放的是主机2的锁，而不是主机1的锁。

#### 2.加锁与解锁的Redis命令分析

```sh
# 可以使用set和del实现加锁和解锁
set key value [NX|XX] [GET] [EX seconds|PX milliseconds|EXAT unix-time-seconds|PXAT unix-time-milliseconds|KEEPTTL]
# NX：如果键不存在，则设置键值对；XX：如果键存在，则设置键值对；GET：返回上次的值；EX：设置键值对过期时间，单位为秒；PX：设置键值对过期时间，单位为毫秒；EXAT：设置键值对过期时间，指定时间戳；PXAT：设置键值对过期时间

# 如果flag不存在则设置为1，过期时间为60s
setnx flag 1 ex 60
# 如果多个线程用分布式竞争同一个资源，由于加入了nx参数，只有一个线程能成功设置，表示这个线程抢占到分布式锁，其他线程会返回0，表示设置失败。
# set命令后用了ex参数指定了flag的生成时间，即使抢占到分布式锁的机器因为故障而无法发起del命令而实现解锁动作，该flag键也能在生存时间过期后自动删除，这样该线程对资源的占有就会自动释放，其他线程继续抢占。
# 占有资源的线程使用完毕后可以通过del flag 命令释放锁，但需要注意的是，加锁和解锁的是同一台机器或同一个线程，避免解错锁。
```

#### 3.Java实现Redis分布式锁

```java
public class RedisLockUtil {
    /**
     * 尝试获取分布式锁
     * 
     * @param jedis Jedis实例，用于操作Redis
     * @param key 锁的键名，通常为加锁的资源标识
     * @param sourceId 请求加锁的唯一标识，如当前线程ID
     * @param expireSeconds 锁的超时时间，单位为秒
     * @return 返回true表示成功获取锁，返回false表示获取锁失败
     */
    public static boolean tryGetDistributedLock(Jedis jedis, String key, String sourceId, int expireSeconds) {
        // 使用set命令的nx（仅当key不存在时设置）和ex（设置过期时间）选项来尝试获取分布式锁
        // 如果锁获取成功，返回"OK"，否则返回null
        // 注意：set nx和设置过期时间expire必须写到一行命令，否则无法保证两个操作的原子性，导致始终无法释放锁
        if (Objects.equals(jedis.set(key, sourceId, SetParams.setParams().nx().ex(expireSeconds)), "OK")) {
            System.out.println(sourceId + " get distributed lock");
            return true;
        }
        return false;
    }

    /**
     * 释放分布式锁
     * 
     * @param jedis Jedis实例，用于执行Redis操作
     * @param key 锁的键名，用于标识一个分布式锁
     * @param sourceId 锁的拥有者标识，用于确保只有锁的拥有者才能释放锁
     * @return boolean 表示锁释放的结果，true表示成功释放，false表示释放失败
     * 
     * 通过使用Lua脚本在Redis服务器端校验并释放锁，以保证操作的原子性
     * 脚本逻辑：检查锁的拥有者是否与传入的sourceId一致，如果一致则删除锁，返回1；否则返回0
     * 这样可以防止其他客户端误释放了不属于自己的锁
     */
    public static boolean releaseDistributedLock(Jedis jedis, String key, String sourceId) {
        // 定义Lua脚本，用于在Redis服务器端检查并释放分布式锁
        String script = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";
        // 执行Lua脚本，传入锁的键名和拥有者标识
        Object result = jedis.eval(script, 1, key, sourceId);
        // 检查脚本执行结果，如果删除成功，则打印信息并返回true
        if ("1".equals(result.toString())) {
            System.out.println(sourceId + " release distributed lock");
            return true;
        }
        // 删除失败，返回false
        return false;
    }
}


public class DistributedRedisLockDemo extends Thread {
    private static final String ACCOUNT_ID = "1234";

    /**
     * 程序入口主函数
     * 该函数的目的是模拟并发环境下对分布式Redis锁的需求
     * 通过创建并启动5个DistributedRedisLockDemo实例来模拟并发场景
     * 每个实例启动一个线程尝试获取分布式锁，从而验证分布式锁在并发环境下的有效性和稳定性
     */
    public static void main(String[] args) {
        for (int i = 0; i < 5; i++) {
            new DistributedRedisLockDemo().start();
        }
    }


    /**
     * 设置账户信息的方法
     * 此方法使用分布式锁来确保并发环境下账户信息的设置操作能够安全执行
     */
    public void setAccount() {
        // 获取当前线程的名称，用作锁的标识
        String sourceId = Thread.currentThread().getName();
        // 初始化锁获取标志为false
        boolean lockFlag = false;
        // 创建Jedis实例，连接本地Redis服务器
        Jedis jedis = new Jedis("127.0.0.1", 6379);
        try {
            // 尝试获取分布式锁，如果成功则执行业务逻辑，否则执行其他操作
            lockFlag = RedisLockUtil.tryGetDistributedLock(jedis, ACCOUNT_ID, sourceId, 10);
            if (lockFlag) {
                // 获取锁成功，执行业务逻辑
                System.out.println(sourceId + "获取锁成功，执行业务逻辑");
            } else {
                // 获取锁失败，执行其他操作
                System.out.println(sourceId + "获取锁失败");
            }
        } catch (Exception ex) {
            System.out.println(ex);
        } finally {
            // 如果成功获取了锁，则在业务逻辑执行完毕后释放锁
            if (lockFlag) {
                RedisLockUtil.releaseDistributedLock(jedis, ACCOUNT_ID, sourceId);
                System.out.println(sourceId + "释放锁成功");
            }
        }
    }

    @Override
    public void run() {
        try {
            setAccount();
        } catch (Exception e) {
            System.out.println(e);
        }
    }
}

// 运行结果：
Thread-0获取锁失败
Thread-3获取锁失败
Thread-1 get distributed lock
Thread-2获取锁失败
Thread-4获取锁失败
Thread-1获取锁成功，执行业务逻辑
Thread-1 release distributed lock
Thread-1释放锁成功
```

#### 三、 java实现Redis限流
> 限流的含义是指在给定的时间内，对某一个资源进行访问的次数不能超过一定的阈值，否则就拒绝访问。比如秒杀模块中，需要确保10s内发往支付模块的请求数量小于500个。限流的作用就是防止某个时间段内的请求数过多，造成模块因高并发而不可用。

#### 1.zset有序集合相关命令与限流
有序集合（zset）是一种特殊类型的集合，它包含一个或多个成员，每个成员都关联有一个分数（score）。有序集合的成员按分数值从小到大排列。有序集合的成员是唯一的，分数可以是整数也可以是浮点数。

```sh
# 向有序集合添加元素
# 在限流场景中，可以将score存储为时间戳，这样zset就可以根据时间戳的score排序
zadd key score member


# 删除键为key，score在min和max之间的所有元素
# 通过这个操作，可以排除限流时间范围外的数据，在此基础上，通过zcard统计有序集合中元素的数量
zremrangeByScore key min max

# 获取键为key的zset中元素的数量
zcard key
```

#### 2.java实现zset限流

```java
public class LimitRequestDemo {
    /**
     * 主函数，用于演示如何使用Jedis操作Redis进行限流控制
     * 该函数首先连接到本地的Redis服务器，然后删除已有的请求类型键，
     * 接着循环5次，每次调用canVisit方法来模拟一次访问请求，并进行限流判断
     * 
     * @param args 命令行参数，本例中未使用
     */
    public static void main(String[] args) {
        // 创建Jedis实例，连接本地Redis服务器
        Jedis jedis = new Jedis("localhost", 6379);
        // 定义请求类型为支付请求
        String requestType = "PayRequest";
        // 删除现有的请求类型键，以确保开始时环境干净
        jedis.del(requestType);
        // 循环5次，模拟5次访问请求
        for (int i = 0; i < 5; i++) {
            // 调用canVisit方法，传入Jedis实例，请求类型，资源限制（100），以及时间窗口（3秒）
            canVisit(jedis, requestType, 100, 3);
        }
    }


    /**
     * 判断是否可以访问
     * 使用Redis的有序集合zset来记录请求的时间戳，通过清除超过时间窗口的请求记录来限制请求频率。
     * 如果在指定时间窗口内的请求数量超过限制，则提示请求过于频繁。
     *
     * @param jedis       Jedis实例，用于操作Redis数据库
     * @param requestType 请求类型，用于区分不同类型的请求并分别限流
     * @param limitTime   单位为秒的时间窗口长度，用于定义多久算作一次请求
     * @param limitCount  在指定时间窗口内允许的最大请求数量
     */
    public static void canVisit(Jedis jedis, String requestType, int limitTime, int limitCount) {
        // 获取当前时间戳
        long currentTimeMillis = System.currentTimeMillis();
        // 将当前时间戳添加到指定请求类型的有序集合中
        jedis.zadd(requestType, currentTimeMillis, Long.toString(currentTimeMillis));
        // 移除超过时间窗口的请求时间戳
        jedis.zremrangeByScore(requestType, 0, currentTimeMillis - limitTime * 1000);
        // 获取当前时间窗口内的请求数量
        Long totalCount = jedis.zcard(requestType);
        // 为请求类型的集合设置过期时间，防止数据溢出
        jedis.expire(requestType, limitTime + 1);
        // 判断请求数是否超过限制
        if (totalCount > limitCount) {
            System.out.println("请求过于频繁，请稍后再试");
        } else {
            System.out.println("请求成功");
        }
    }
}

// 运行结果：
请求成功
请求成功
请求成功
请求过于频繁，请稍后再试
请求过于频繁，请稍后再试
```

#### 4.Redis压力测试实战

```sh
 redis-benchmark [option] [optiion value]
 # option是参数项，option value是参数值
 # -h 指定redis服务器地址
 # -p 指定redis服务器端口
 # -c 指定并发连接数
 # -n 指定请求次数  
 # -q 强制退出Redis,显示时只给出“每秒能处理的请求数”这个值
 # -t 指定测试的命令，默认执行所有命令

PS D:\code\blogs\farb.github.io> docker exec -it redisDemo bash

# 对本机6379端口的redis服务器进行压测，执行set和get命令各2000次
root@563f5a099d06:/data# redis-benchmark -h 127.0.0.1 -p 6379 -t set,get -n 2000

# 2000次请求set的压测结果，0.01s处理完成
====== SET ======
  2000 requests completed in 0.01 seconds
  50 parallel clients
  3 bytes payload
  keep alive: 1
  host configuration "save": 3600 1 300 100 60 10000
  host configuration "appendonly": no
  multi-thread: no

# 显示了n%的请求处理时间
Latency by percentile distribution:
0.000% <= 0.039 milliseconds (cumulative count 1)
50.000% <= 0.095 milliseconds (cumulative count 1043)
75.000% <= 0.111 milliseconds (cumulative count 1677)
87.500% <= 0.119 milliseconds (cumulative count 1775)
93.750% <= 0.167 milliseconds (cumulative count 1887)
96.875% <= 0.207 milliseconds (cumulative count 1942)
98.438% <= 0.247 milliseconds (cumulative count 1976)
99.219% <= 0.271 milliseconds (cumulative count 1986)
99.609% <= 0.295 milliseconds (cumulative count 1994)
99.805% <= 0.303 milliseconds (cumulative count 1997)
99.902% <= 0.319 milliseconds (cumulative count 1999)
99.951% <= 0.327 milliseconds (cumulative count 2000)
100.000% <= 0.327 milliseconds (cumulative count 2000)

Cumulative distribution of latencies:
73.050% <= 0.103 milliseconds (cumulative count 1461)
97.100% <= 0.207 milliseconds (cumulative count 1942)
99.850% <= 0.303 milliseconds (cumulative count 1997)
100.000% <= 0.407 milliseconds (cumulative count 2000)

# 汇总信息，吞吐量：每秒28.6万次请求
Summary:
  throughput summary: 285714.28 requests per second
  latency summary (msec):
          avg       min       p50       p95       p99       max
        0.103     0.032     0.095     0.175     0.255     0.327

#以下时get的压测结果，与上面类似
====== GET ======
  2000 requests completed in 0.01 seconds
  50 parallel clients
  3 bytes payload
  keep alive: 1
  host configuration "save": 3600 1 300 100 60 10000
  host configuration "appendonly": no
  multi-thread: no

Latency by percentile distribution:
0.000% <= 0.039 milliseconds (cumulative count 2)
50.000% <= 0.103 milliseconds (cumulative count 1211)
75.000% <= 0.119 milliseconds (cumulative count 1512)
87.500% <= 0.159 milliseconds (cumulative count 1751)
93.750% <= 0.223 milliseconds (cumulative count 1878)
96.875% <= 0.407 milliseconds (cumulative count 1938)
98.438% <= 0.759 milliseconds (cumulative count 1969)
99.219% <= 0.935 milliseconds (cumulative count 1985)
99.609% <= 0.967 milliseconds (cumulative count 1993)
99.805% <= 0.999 milliseconds (cumulative count 1997)
99.902% <= 1.071 milliseconds (cumulative count 1999)
99.951% <= 1.079 milliseconds (cumulative count 2000)
100.000% <= 1.079 milliseconds (cumulative count 2000)

Cumulative distribution of latencies:
60.550% <= 0.103 milliseconds (cumulative count 1211)
92.650% <= 0.207 milliseconds (cumulative count 1853)
96.450% <= 0.303 milliseconds (cumulative count 1929)
96.900% <= 0.407 milliseconds (cumulative count 1938)
97.250% <= 0.503 milliseconds (cumulative count 1945)
97.650% <= 0.607 milliseconds (cumulative count 1953)
98.000% <= 0.703 milliseconds (cumulative count 1960)
98.600% <= 0.807 milliseconds (cumulative count 1972)
98.950% <= 0.903 milliseconds (cumulative count 1979)
99.900% <= 1.007 milliseconds (cumulative count 1998)
100.000% <= 1.103 milliseconds (cumulative count 2000)

Summary:
  throughput summary: 249999.98 requests per second
  latency summary (msec):
          avg       min       p50       p95       p99       max
        0.131     0.032     0.103     0.247     0.919     1.079
```

再执行一个压测命令查看一下：

```sh
# 20个并发连接，2000次请求，没有指定-t参数，默认执行所有命令，使用-q参数，只显示每秒处理的请求数，如果去掉-q，则会显示每条命令的详细信息。
root@563f5a099d06:/data# redis-benchmark -h localhost -p 6379 -c 20 -n 2000 -q

# 显示每秒处理的请求数，以及50%的请求处理耗时
PING_INLINE: 285714.28 requests per second, p50=0.047 msec
PING_MBULK: 285714.28 requests per second, p50=0.047 msec
SET: 249999.98 requests per second, p50=0.047 msec
GET: 249999.98 requests per second, p50=0.047 msec
INCR: 249999.98 requests per second, p50=0.047 msec
LPUSH: 285714.28 requests per second, p50=0.039 msec
RPUSH: 249999.98 requests per second, p50=0.047 msec
LPOP: 222222.23 requests per second, p50=0.039 msec
RPOP: 285714.28 requests per second, p50=0.047 msec
SADD: 249999.98 requests per second, p50=0.047 msec
HSET: 249999.98 requests per second, p50=0.047 msec
SPOP: 222222.23 requests per second, p50=0.047 msec
ZADD: 222222.23 requests per second, p50=0.047 msec
ZPOPMIN: 222222.23 requests per second, p50=0.047 msec
LPUSH (needed to benchmark LRANGE): 249999.98 requests per second, p50=0.047 msec
LRANGE_100 (first 100 elements): 90909.09 requests per second, p50=0.119 msec
LRANGE_300 (first 300 elements): 35714.29 requests per second, p50=0.279 msec
LRANGE_500 (first 500 elements): 25000.00 requests per second, p50=0.391 msec
LRANGE_600 (first 600 elements): 21505.38 requests per second, p50=0.455 msec
MSET (10 keys): 333333.34 requests per second, p50=0.039 msec
XADD: 285714.28 requests per second, p50=0.039 msec
```
