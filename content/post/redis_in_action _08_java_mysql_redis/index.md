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

### 2. Java操作各种Redis数据类型
#### 读写列表类对象
```java
    public static void main(String[] args) {
        listDemo();
    }

    /**
     * List数据结构演示
     * 该方法用于演示如何使用Jedis操作List数据结构，包括插入和查询操作
     */
    private static void listDemo() {
        // 创建Jedis对象，连接本地Redis服务器
        Jedis jedis = new Jedis("localhost", 6379);
        
        // 向List的尾部添加元素
        jedis.rpush("001", "Peter");
        jedis.rpush("001", "15");
        jedis.rpush("001", "Male");
        
        // 检查Key是否存在
        if (jedis.exists("001")) {
            // 如果Key存在，打印List中的每个元素
            System.out.println(jedis.lindex("001", 0));
            System.out.println(jedis.lindex("001", 1));
            System.out.println(jedis.lindex("001", 2));
        } else {
            // 如果Key不存在，打印提示信息
            System.out.println("Key does not exist,get from db");
        }
    }
// 运行程序，输出结果如下：
Peter
15
Male
```

#### 2.读写哈希表对象
> 如果待缓存对象的属性个数很多，就建议用哈希表对象来缓存：一方面，在哈希表中是用属性名称来定位数据的，而不是属性的顺序；另一方面，哪怕没存若干属性，也不会影响对该对象的读取，顶多就是读取的属性值为空。

```java
    /**
     * 示例如何使用Jedis操作Hash数据类型
     * 该方法连接到本地的Redis服务器，使用HSET命令向两个员工ID（Emp002和Emp003）添加信息，
     * 并使用HGET命令获取这些信息
     */
    static void hashDemo() {
        // 创建Jedis实例，连接到本地Redis服务器
        Jedis jedis = new Jedis("localhost", 6379);
        
        // 向Emp002这个Hash中设置name、age和gender字段
        jedis.hset("Emp002", "name", "Peter");
        jedis.hset("Emp002", "age", "15");
        jedis.hset("Emp002", "gender", "Male");
        
        // 获取并打印Emp002的name和age字段
        System.out.println(jedis.hget("Emp002", "name"));
        System.out.println(jedis.hget("Emp002", "age"));

        // 使用Map封装Emp003的信息，然后一次性设置到Redis中
        Map<String, String> emp03 = new HashMap<>();
        emp03.put("name", "farb");
        emp03.put("age", "18");
        emp03.put("gender", "Male");
        jedis.hset("Emp003", emp03);

        // 再次获取并打印Emp002和Emp003的name字段，展示数据一致性
        System.out.println(jedis.hget("Emp002", "name"));
        System.out.println(jedis.hget("Emp002", "age"));
    }
// 运行程序，输出结果如下：
Peter
15
farb
18
```

#### 3.读写集合对象
> 一般项目中，大多使用Redis的字符串，列表和哈希表来缓存数据，而集合对象一般用于去重的场景。

```java
    /**
     * 使用Jedis操作Redis的Set数据结构的演示方法
     * 该方法展示了如何向Set中添加元素，并打印出Set中的所有元素
     */
    static void setDemo() {
        // 创建Jedis对象，连接本地的Redis服务器
        Jedis jedis = new Jedis("localhost", 6379);
        // 向Set“bonusId”中添加元素“1”、“2”、“3”、“4”、“1”，注意重复元素会被忽略
        jedis.sadd("bonusId", "1", "2", "3", "4", "1");
        // 打印Set“bonusId”中的所有元素
        System.out.println(jedis.smembers("bonusId"));
    }
// 运行程序，输出结果如下：
[1, 2, 3, 4]
```

#### 4.读写有序集合对象

```java
    /**
     * 示范使用Sorted Set数据结构的操作
     * 该方法主要展示了如何在Redis中使用Jedis操作Sorted Set，包括添加成员和获取排序范围
     */
    static void sortedSetDemo() {
        // 创建Jedis对象，连接本地的Redis服务器
        Jedis jedis = new Jedis("localhost", 6379);
        // 向Sorted Set "emps"中添加成员"Peter"，分数为1.0
        jedis.zadd("emps", 1.0, "Peter");
        // 向Sorted Set "emps"中添加成员"Jane"，分数为3.0
        jedis.zadd("emps", 3.0, "Jane");
        // 向Sorted Set "emps"中添加成员"Mike"，分数为2.0
        jedis.zadd("emps", 2.0, "Mike");
        // 打印Sorted Set "emps"中所有成员，验证添加操作的结果
        System.out.println(jedis.zrange("emps", 0, -1));

        // 根据分数范围查询并打印Sorted Set "emps"中的成员及其分数，分数范围为2.0到3.0
        System.out.println(jedis.zrangeByScoreWithScores("emps", 2.0, 3.0));
    }
// 运行程序，输出结果如下：
[Peter, Mike, Jane]
[[Mike,2.0], [Jane,3.0]]
```

#### 5.操作地理位置数据

```java
    /**
     * 地理位置演示函数
     * 该函数演示了如何使用Jedis操作Redis中的地理位置数据
     * 主要包括添加地理位置数据、获取位置信息、查询一定半径内的地理位置以及计算两点间的距离
     */
    static void geoDemo() {
        // 创建Jedis对象，连接本地的Redis服务器
        Jedis jedis = new Jedis("localhost", 6379);
        
        // 向Redis中添加地理位置数据，包括经度、纬度和位置标识
        jedis.geoadd("geo", 120.52, 30.40, "pos1");
        jedis.geoadd("geo", 120.52, 31.53, "pos2");
        jedis.geoadd("geo", 122.12, 30.40, "pos3");
        jedis.geoadd("geo", 122.12, 31.53, "pos4");
        
        // 打印所有位置的地理坐标
        System.out.println(jedis.geopos("geo", "pos1", "pos2", "pos3", "pos4"));
        
        // 查询以给定经纬度为中心，指定半径内的地理位置
        List<GeoRadiusResponse> geoList = jedis.georadius("geo", 120.52, 30.40, 200, GeoUnit.KM);
        for (GeoRadiusResponse res : geoList) {
            // 打印查询到的地理位置
            System.out.println(res.getMemberByString());
        }
        
        // 计算两个地理位置之间的距离
        Double distance = jedis.geodist("geo", "pos1", "pos2", GeoUnit.KM);
        // 打印计算得到的距离
        System.out.println(distance);
    }
// 运行程序，输出结果如下：
[(120.5200007557869,30.399999526689975), (120.5200007557869,31.530001032013715), (122.11999744176865,30.399999526689975), (122.11999744176865,31.530001032013715)]
pos1
pos2
pos3
pos4
125.6859

```
