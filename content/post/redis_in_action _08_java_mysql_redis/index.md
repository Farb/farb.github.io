---
title: 基于Docker的Redis实战--Java整合Mysql和Redis
description:
slug: redis_in_action _08_java_mysql_redis
date: 2024-08-14
image: 
categories:
    - Redis
tags:
    - redis
    - docker
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---
源码链接： https://gitee.com/farb/architect-practicer-code 
### 1. Java通过Jedis读写Redis
#### 1.Maven引入Jedis依赖
IDEA新建项目RedisBasedDocker，pom.xml文件内容如下，引入Redis的客户端Jedis依赖，使用java11版本

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>top.farb</groupId>
    <artifactId>RedisBasedDocker</artifactId>
    <version>1.0-SNAPSHOT</version>

    <dependencies>
        <dependency>
            <groupId>redis.clients</groupId>
            <artifactId>jedis</artifactId>
            <version>3.3.0</version>
        </dependency>
    </dependencies>
    <properties>
        <maven.compiler.source>11</maven.compiler.source>
        <maven.compiler.target>11</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

</project>
```

#### 2.通过Jedis读写Redis字符串

```sh
# 首先通过docker的方式启动redis服务器
PS D:\code\architect-practicer-code\java\RedisBasedDocker> docker run -itd --name javaRedis -p 6379:6379 redis:latest
```

```java
    public static void main(String[] args) {
        // 创建一个Jedis对象，连接到本地的Redis服务器
        Jedis jedis = new Jedis("localhost", 6379);

        // 验证与Redis服务器的连接是否成功
        System.out.println("jedis.ping() = " + jedis.ping());

        // 设置键值对，将"name"键的值设置为"farb"
        System.out.println(jedis.set("name", "farb"));

        // 获取"name"键的值，并打印
        System.out.println(jedis.get("name"));
        // 尝试获取一个不存在的键，将返回null
        System.out.println(jedis.get("notExistKey"));
    }

// 运行程序，输出结果如下：
jedis.ping() = PONG
OK
farb
null
```

#### 3.操作各种Redis命令
```java
    /**
     * 演示Redis中一些其他命令的使用
     * 本函数主要展示如何在Redis中执行如设置键值对、检查键是否存在、匹配键名和删除键等操作
     */
    static void otherCommands() {
        // 创建Jedis实例，连接到本地Redis服务器
        Jedis jedis = new Jedis("localhost", 6379);

        // 设置键值对，为键"name"、"age"和"salary"分别设置值"farb"、"18"和"30000"
        jedis.set("name", "farb");
        jedis.set("age", "18");
        jedis.set("salary", "30000");

        // 检查键"name"是否存在于Redis中，并打印结果
        System.out.println(jedis.exists("name"));
        // 检查键"notExistKey"是否存在于Redis中，并打印结果
        System.out.println(jedis.exists("notExistKey"));

        // 匹配所有包含"a"的键，并打印结果
        System.out.println(jedis.keys("*a*"));
        // 删除键"name"，并打印删除的键的数量
        System.out.println(jedis.del("name"));
        // 再次匹配所有包含"a"的键，并打印结果，以验证删除操作的效果
        System.out.println(jedis.keys("*a*"));
    }

// 运行程序，输出结果如下：
true
false
[name, salary, age]
1
[age, salary]
```

#### 4.以事务的方式操作redis
> jedis.multi()方法用于创建一个事务，然后通过调用jedis.exec()方法来执行这个事务，jedis.discard()方法用于取消这个事务，jedis.watch()方法用于对一个键进行监视，jedis.unwatch()方法用于取消对一个键的监视。watch的作用相当于多线程中的加锁操作，需要unwatch释放锁。
```java
    /**
     * 本方法用于演示Jedis事务的基本使用和特性
     * 事务可以看作是一个命令的集合，这些命令在发送到服务器时会被当作一个整体来处理
     * 这个方法展示了事务的创建、命令添加、取消和执行过程
     */
    static void transactionDemo() {
        // 创建一个Jedis实例，连接到本地的Redis服务器
        Jedis jedis = new Jedis("localhost", 6379);

        // 开始一个事务
        Transaction transaction = jedis.multi();
        // 在事务中设置两个键值对
        transaction.set("trans1", "1");
        transaction.set("trans2", "2");
        // 取消这个事务，事务中的所有命令都不会被执行
        transaction.discard();

        // 打印出所有以"trans"开头的键，验证事务是否成功取消
        System.out.println(jedis.keys("trans*"));

        // 设置一个标记键，用于演示watch命令
        jedis.set("flag", "true");
        // 对"flag"键进行监视，以便在事务中使用
        jedis.watch("flag");

        // 再次开始一个事务
        Transaction transaction1 = jedis.multi();
        // 在事务中设置两个键值对
        transaction1.set("key1", "1");
        transaction1.set("key2", "2");
        // 执行这个事务，事务中的所有命令都会被执行
        transaction1.exec();
        // 打印出所有以"key"开头的键，验证事务是否成功执行
        System.out.println(jedis.keys("key*"));

        // 取消对"flag"键的监视
        jedis.unwatch();
    }

// 运行程序，输出结果如下：
[]
[key1, key2]
```

#### 5.Jedis连接池
> 如果多个客户端在使用Redis时，需要频繁的创建和销毁Jedis实例，这会消耗大量的资源，导致性能下降。为了解决这个问题，Redis提供了连接池的功能，通过连接池可以复用已经创建的连接，从而提高Redis的性能。
> 引入了Jedis连接池，大量客户端不是直接向Redis服务器申请连接，而是从连接池中获取连接，并在使用完毕后将其归还给连接池，这样可以避免连接的频繁创建和销毁，高并发场景里建议使用连接池，从而提高Redis的性能。
```java
    /**
     * 配置并使用Jedis连接池来管理Redis连接
     * 该方法演示了如何创建Jedis连接池并配置其参数，然后使用该连接池获取Jedis实例进行Redis操作，
     * 最后关闭连接池
     */
    static void jedisPool() {
        // 创建Jedis连接池配置对象
        JedisPoolConfig jedisPoolConfig = new JedisPoolConfig();
        // 设置连接池最大连接数
        jedisPoolConfig.setMaxTotal(10);
        // 设置连接池最大空闲连接数
        jedisPoolConfig.setMaxIdle(5);
        // 设置连接池最小空闲连接数
        jedisPoolConfig.setMinIdle(3);
        // 设置获取连接的最大等待时间
        jedisPoolConfig.setMaxWaitMillis(1000);
        // 设置当连接池资源耗尽时是否阻塞等待
        jedisPoolConfig.setBlockWhenExhausted(false);

        // 根据配置创建Jedis连接池
        JedisPool jedisPool = new JedisPool(jedisPoolConfig, "localhost", 6379);
        // 从连接池中获取Jedis实例
        Jedis jedis = jedisPool.getResource();
        // 使用Jedis实例执行set操作
        jedis.set("name", "farb");
        // 使用Jedis实例执行get操作并打印结果
        System.out.println(jedis.get("name"));
        // 关闭连接池，释放资源
        jedisPool.close();
    }

```

#### 6.用管道的方式提升操作性能
> 在Redis中，管道是一种优化Redis操作性能的技术，它允许客户端一次性发送多个命令，然后一次性从Redis服务器接收多个响应。通过使用管道，客户端可以减少网络延迟，从而提高操作性能。在大数据的操作场景里，通过管道的方式能大量节省“命令和结果的传输时间”，从而提升性能。

```java
    /**
     * 演示使用Redis的Jedis客户端进行数据操作时，使用pipeline技术的性能提升
     * 通过对比使用和不使用pipeline的方式进行数据设置和获取操作的耗时，来展示pipeline的优势
     */
    static void pipelineDemo() {
        // 创建Jedis实例，连接本地Redis服务器
        Jedis jedis = new Jedis("localhost", 6379);
        
        // 记录开始时间
        long start = System.currentTimeMillis();
        
        // 不使用pipeline进行10000次set和get操作
        for (int i = 0; i < 10000; i++) {
            jedis.set("key" + i, "value" + i);
            jedis.get("key" + i);
        }
        
        // 记录结束时间，计算不使用pipeline时的操作耗时
        long end = System.currentTimeMillis();
        System.out.println("不使用管道耗时：" + (end - start) + "ms");

        // 重新记录开始时间，准备使用pipeline
        start = System.currentTimeMillis();
        
        // 创建Pipeline对象，用于批量执行Jedis命令
        Pipeline pipeline = jedis.pipelined();
        
        // 使用pipeline进行10000次set和get操作
        for (int i = 0; i < 10000; i++) {
            pipeline.set("key" + i, "value" + i);
            pipeline.get("key" + i);
        }
        
        // 同步执行pipeline中积累的所有命令
        pipeline.sync();
        
        // 记录结束时间，计算使用pipeline时的操作耗时
        end = System.currentTimeMillis();
        System.out.println("使用管道耗时：" + (end - start) + "ms");
    }

// 运行程序，输出结果如下：
不使用管道耗时：6141ms
使用管道耗时：28ms
```
**通过以上例子，可以看到，如果需要大批量的向Redis服务器读写数据，那么建议采用管道的方式提升性能。**
