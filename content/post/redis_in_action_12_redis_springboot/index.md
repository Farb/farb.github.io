---
title: 基于Docker的Redis实战-Redis与SpringBoot整合
description:
slug: redis_in_action_12_redis_springboot
date: 2024-11-22
image: 
categories:
    - Redis
tags:
    - redis
    - docker
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---
源码链接： https://gitee.com/farb/architect-practicer-code 

## 在SpringBoot框架引入Redis

### 准备MySQL数据库和数据表

1. 创建MySQL的镜像容器并进入容器

```bash
PS C:\Users\farbg> docker run -itd -p 3306:3306 --name mysqlSpringBoot -e MYSQL_ROOT_PASSWORD=123456 mysql
324a64cbc2c9b36dca5f08fd86f301bd6cf55b6fcfa403536d538537e660c68c

PS C:\Users\farbg> docker exec -it mysqlSpringBoot bash
```

2. 登录mysql数据库，并创建数据库和数据表
   
```bash
bash-5.1# mysql -u root -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 9
Server version: 8.4.0 MySQL Community Server - GPL

Copyright (c) 2000, 2024, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> use stockDB;
Database changed
mysql> create table stock(id int not null primary key,name varchar(20),num int);
Query OK, 0 rows affected (0.02 sec)

# 查看数据表结构
mysql> desc stock;
+-------+-------------+------+-----+---------+-------+
| Field | Type        | Null | Key | Default | Extra |
+-------+-------------+------+-----+---------+-------+
| id    | int         | NO   | PRI | NULL    |       |
| name  | varchar(20) | YES  |     | NULL    |       |
| num   | int         | YES  |     | NULL    |       |
+-------+-------------+------+-----+---------+-------+
3 rows in set (0.00 sec)

# 插入数据
mysql> insert into stock(id,name,num) values (1,'Computer',10);
Query OK, 1 row affected (0.00 sec)

# 查看数据
mysql> select * from stock;
+----+----------+------+
| id | name     | num  |
+----+----------+------+
|  1 | Computer |   10 |
+----+----------+------+
1 row in set (0.00 sec)
```

### 搭建SpringBoot项目

1. 新建一个Maven项目SpringBootForRedis，并添加依赖如下:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://maven.apache.org/POM/4.0.0"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.4.0</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>blog.farb.top</groupId>
    <artifactId>SpringBootForRedis</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>SpringBootForRedis</name>
    <description>SpringBootForRedis</description>
    <properties>
        <java.version>17</java.version>
    </properties>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>com.alibaba.fastjson2</groupId>
            <artifactId>fastjson2</artifactId>
            <version>2.0.43</version>
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>8.0.33</version>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.18.34</version>
            <scope>compile</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>

```

2. 代码结构目录如下：

```bash
│  SpringBootForRedisApplication.java
│
├─controller
│      StockController.java
│
├─entity
│      Stock.java
│
├─repo
│      StockRedisRepo.java
│      StockRepo.java
│
└─service
        StockService.java
        StockServiceImpl.java

```

StockController.java

```java
package blog.farb.top.springbootforredis.controller;

import blog.farb.top.springbootforredis.entity.Stock;
import blog.farb.top.springbootforredis.service.StockService;
import jakarta.annotation.Resource;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class StockController {

    @Resource
    private StockService stockService;

    @GetMapping("/stock/{id}")
    public Stock getStockById(@PathVariable int id) {
        return stockService.getStockById(id);
    }
}

```

StockServiceImpl.java

```java
package blog.farb.top.springbootforredis.service;

import blog.farb.top.springbootforredis.entity.Stock;
import blog.farb.top.springbootforredis.repo.StockRedisRepo;
import blog.farb.top.springbootforredis.repo.StockRepo;
import jakarta.annotation.Resource;
import org.springframework.stereotype.Service;

@Service
public class StockServiceImpl implements StockService {
    private static final String REDIS_KEY_PREFIX = "stock:";

    @Resource
    private StockRedisRepo stockRedisRepo;

    @Resource
    private StockRepo stockRepo;

    @Override
    public Stock getStockById(int id) {
        String key = REDIS_KEY_PREFIX + id;
        Stock stock = stockRedisRepo.getStockById(key);
        if (stock != null) {
            System.out.println("Get stock from Redis");
            return stock;
        }
        System.out.println("Get stock from DB");
        stock = stockRepo.findById(id).orElse(null);
        stockRedisRepo.addStock(key, stock, 1000 * 60 * 10);
        return stock;
    }
}

```

StockRedisRepo.java

```java
package blog.farb.top.springbootforredis.repo;

import blog.farb.top.springbootforredis.entity.Stock;
import com.alibaba.fastjson2.JSON;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Repository;

@Repository
public class StockRedisRepo {
    @Autowired
    private RedisTemplate<String, String> redisTemplate;

    public Stock getStockById(String key) {
        String jsonStr = redisTemplate.opsForValue().get(key);
        return JSON.parseObject(jsonStr, Stock.class);
    }

    public void addStock(String key, Stock stock, int expireSeconds) {
        redisTemplate.opsForValue().set(key, JSON.toJSONString(stock), expireSeconds);
    }
}

```

StockRepo.java

```java
package blog.farb.top.springbootforredis.repo;

import blog.farb.top.springbootforredis.entity.Stock;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.util.Optional;

@Repository
public interface StockRepo extends JpaRepository<Stock, Integer> {
    Optional<Stock> findById(int id);
}

```

Stock.java

```java
package blog.farb.top.springbootforredis.entity;

import jakarta.persistence.Entity;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import lombok.Data;

@Entity
@Data
public class Stock {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;

    private String name;

    private String num;
}

```

运行结果可知，首次先从数据库查询，然后把数据存入Redis，第二次直接从Redis中获取数据。

```bash
Get stock from DB
Hibernate: select s1_0.id,s1_0.name,s1_0.num from stock s1_0 where s1_0.id=?
Get stock from Redis
```

## 在SpringBoot中实现秒杀案例

```java
BuyController.java

@RestController
public class BuyController {

    private final SellService sellService;

    public BuyController(SellService sellService) {
        this.sellService = sellService;
    }

    @GetMapping("/quickBuy/{item}/{buyer}")
    public String quickBuy(@PathVariable String item, @PathVariable String buyer) {
        String res = sellService.quickBuy(item, buyer);
        return res.equals("1") ? buyer + " Success" : buyer + "Fail";
    }

    @GetMapping("/quickBuy/benchMark")
    public String benchMark() {
        for (int i = 0; i < 15; i++) {
            new BenchMarkTest().start();
        }
        return "Success";
    }
}


SellService.java

@Service
public class SellService {
    private final RedisTemplate redisTemplate;

    public SellService(RedisTemplate redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    public String quickBuy(String item, String buyer) {
        String luaScript = """
                local buyer=ARGV[1];
                local productName=KEYS[1];
                local leftCount=tonumber(redis.call('get',productName));
                if(leftCount<=0) then
                	return 0
                else
                	redis.call('decrby',productName,1);
                	redis.call('rpush',"buyerList",buyer)
                	return 1
                end 
                                """;
        // 如果这里不设置序列化器，会使用默认的Jdk会序列器，导致key和value有乱码,而且会导致上面的lua脚本查询不到数据。
        // 当然这里的代码可以移到配置文件中                        
        redisTemplate.setKeySerializer(new StringRedisSerializer());
        redisTemplate.setValueSerializer(new StringRedisSerializer());

        // 注意，这里必须使用Long返回类型，因为redis中的Integer对应java中的Long
        RedisScript<Long> defaultRedisScript = new DefaultRedisScript<>(luaScript, Long.class);
        Object res = redisTemplate.execute(defaultRedisScript, Collections.singletonList(item), buyer);
        return res.toString();
    }
}


BenchMarkTest.java

public class BenchMarkTest extends Thread {
    @Override
    public void run() {
        new QuickBuyClient().quickBuy("Gold", "buyer" + Thread.currentThread().getName());
    }
}

QuickBuyClient.java

public class QuickBuyClient {
    private final RestTemplate restTemplate = new RestTemplate();

    public void quickBuy(String item, String buyer) {
        String res = restTemplate.getForObject("http://localhost:8080/quickBuy/{item}/{buyer}", String.class, item, buyer);
        System.out.println("秒杀结果：" + res);
    }
}

```

调用秒杀接口前，先设置一下redis中的数据，如下：

```bash
127.0.0.1:6379> set Gold 10 ex 3600
OK
127.0.0.1:6379> del buyerList
(integer) 1
```

调用模拟秒杀接口 http://localhost:8080/quickBuy/benchMark，测试结果如下：

**可以看到，一共10件库存，但是有15个线程模拟15个人同时去抢，结果只成功了10个，其余5个失败。可见没有发生超卖现象。**

```bash
秒杀结果：buyerThread-24 Fail
秒杀结果：buyerThread-23 Success
秒杀结果：buyerThread-28 Fail
秒杀结果：buyerThread-22 Fail
秒杀结果：buyerThread-18 Success
秒杀结果：buyerThread-30 Success
秒杀结果：buyerThread-27 Success
秒杀结果：buyerThread-25 Success
秒杀结果：buyerThread-16 Fail
秒杀结果：buyerThread-17 Success
秒杀结果：buyerThread-29 Success
秒杀结果：buyerThread-19 Success
秒杀结果：buyerThread-20 Success
秒杀结果：buyerThread-21 Success
秒杀结果：buyerThread-26 Fail
```