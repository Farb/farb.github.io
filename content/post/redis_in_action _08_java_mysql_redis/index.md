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
#### 6.Redis与Mysql的整合
##### 通过Docker安装Mysql开发环境
```sh
# 通过docker pull mysql 拉取镜像并下载到本地后，查看镜像并运行镜像
PS D:\code\blogs\farb.github.io> docker images
REPOSITORY    TAG       IMAGE ID       CREATED         SIZE
mysql         latest    05247af91864   8 weeks ago     578MB
# 通过-e 设置环境变量，指定该容器包含的Mysql服务器登录密码是123456
PS D:\code\blogs\farb.github.io> docker run -itd -p 3306:3306 --name mysql -e MYSQL_ROOT_PASSWORD=123456 mysql
0fab30cacbb5374936d664a694056f2972f0cdeedabe21b969c1be6c62346ca8

# 进入容器，登录后建库建表，插入数据
PS D:\code\blogs\farb.github.io> docker exec -it mysql bash
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

mysql> create database redisDemo;
Query OK, 1 row affected (0.00 sec)

mysql> use redisDemo;
Database changed
mysql> create table student (
    -> id int not null primary key,
    -> name varchar(20),
    -> age int,
    -> score float
    -> );
Query OK, 0 rows affected (0.02 sec)

mysql> insert into student (id,name,age,score) values(1,'Peter',18,100);
Query OK, 1 row affected (0.01 sec)

mysql> insert into student (id,name,age,score) values(2,'Tom',17,90);
Query OK, 1 row affected (0.01 sec)

mysql> insert into student (id,name,age,score) values(3,'John',16,88);
Query OK, 1 row affected (0.01 sec)

# 查看数据,至此，mysql环境搭建完毕
mysql> select * from student; 
+----+-------+------+-------+
| id | name  | age  | score |
+----+-------+------+-------+
|  1 | Peter |   18 |   100 |
|  2 | Tom   |   17 |    90 |
|  3 | John  |   16 |    88 |
+----+-------+------+-------+
3 rows in set (0.00 sec)
```

#### 通过JDBC连接并操作Mysql数据库
沿用上面的pom.xml文件，确保有以下依赖，如果没有添加依赖
```xml
      <dependency>
            <groupId>redis.clients</groupId>
            <artifactId>jedis</artifactId>
            <version>3.3.0</version>
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>8.0.33</version>
        </dependency>
```

```java
    public static void main(String[] args) {
        mysqlDemo();
    }

    /**
     * MySQL演示函数
     * 该函数演示了如何使用Java连接到MySQL数据库，并从数据库中检索数据
     * 它首先加载数据库驱动，然后创建与数据库的连接，执行查询并打印结果
     * 最后，它关闭数据库连接和相关资源
     */
    static void mysqlDemo() {
        // 显示定义MySQL数据库的JDBC驱动，不指定驱动也会自动注册
        String driver = "com.mysql.cj.jdbc.Driver";
        // 定义数据库连接URL
        String url = "jdbc:mysql://localhost:3306/redisDemo";
        // 定义数据库用户名
        String user = "root";
        // 定义数据库密码
        String password = "123456";
        try {
            // 加载并注册JDBC驱动
            Class.forName(driver);
            // 建立与数据库的连接
            Connection conn = java.sql.DriverManager.getConnection(url, user, password);
            // 准备SQL语句，用于查询学生信息
            PreparedStatement stmt = conn.prepareStatement("select * from student");
            // 执行查询并获取结果集
            ResultSet rs = stmt.executeQuery();
            // 遍历结果集，打印每个学生的详细信息
            while (rs.next()) {
                System.out.print("id:" + rs.getString("id") + ",");
                System.out.print("name:" + rs.getString("name") + ",");
                System.out.print("age:" + rs.getInt("age") + ",");
                System.out.println("score:" + rs.getString("score"));
            }
            // 关闭结果集、PreparedStatement和数据库连接，释放资源
            rs.close();
            stmt.close();
            conn.close();
        } catch (Exception e) {
            // 打印异常信息，用于调试和错误追踪
            System.out.println(e);
        }
    }
```

#### 引入Redis做缓存
定义一个和数据库中Student表对应的Student类
```java
import lombok.Data;

@Data
public class Student {
    private int id;
    private String name;
    private int age;
    private Float score;
}

    /**
     * 数据库与Redis示例函数
     * 此函数展示了如何从数据库中获取学生信息，并打印出来
     * 它首先进行初始化，然后尝试从数据库中获取学生信息，并打印学生的ID、姓名、年龄和分数
     * 注意：这个函数假设存在一个初始化函数init()和一个用于从数据库获取学生信息的函数getStudent(int id)
     */
    static void mysqlRedisDemo() {
        init(); // 初始化环境或资源，为数据库操作做准备
        for (int i = 1; i <= 2; i++) { // 循环两次，尝试获取学生信息，这里循环的次数可能基于特定需求或示例目的设定
            Student student = getStudent(1); // 从数据库中获取指定ID为1的学生信息
            if (student != null) { // 检查获取到的学生信息是否为空，不为空则打印信息
                System.out.print("id=" + student.getId() + ",");
                System.out.print("name=" + student.getName() + ",");
                System.out.print("age=" + student.getAge() + ",");
                System.out.println("score=" + student.getScore()); // 打印学生的各项信息
            }
        }
    }

    static void init() {
        // 定义MySQL数据库的JDBC驱动
        String driver = "com.mysql.cj.jdbc.Driver";
        // 定义数据库连接URL
        String url = "jdbc:mysql://localhost:3306/redisDemo";
        // 定义数据库用户名
        String user = "root";
        // 定义数据库密码
        String password = "123456";
        try {
            // 加载并注册JDBC驱动
            Class.forName(driver);
            // 建立与数据库的连接
            conn = java.sql.DriverManager.getConnection(url, user, password);
            jedis = new Jedis("localhost", 6379);
        } catch (Exception e) {
            System.out.println(e);
        }
    }
    
    /**
     * 根据学生ID获取学生信息
     * 首先尝试从Redis缓存中获取学生信息，如果缓存存在，则从Redis中读取并返回学生信息
     * 如果Redis中不存在该学生信息，则从MySQL数据库中查询，并将结果缓存到Redis中
     * 
     * @param id 学生ID
     * @return 返回学生信息，如果找不到对应ID的学生，则返回null
     */
    static Student getStudent(int id) {
        // 构建Redis中的学生信息键名
        String key = "student:" + id;
        // 检查Redis中是否存在该学生信息
        if (jedis.exists(key)) {
            System.out.println("从redis中获取数据");
            // 从Redis中获取学生信息，并映射到Student对象中
            List<String> list = jedis.lrange(key, 0, 2);
            Student student = new Student();
            student.setId(id);
            student.setName(list.get(0));
            student.setAge(Integer.parseInt(list.get(1)));
            student.setScore(Float.parseFloat(list.get(2)));
            return student;
        }
        System.out.println("从mysql中获取数据");
        try {
            // 准备SQL语句，用于查询学生信息
            PreparedStatement stmt = conn.prepareStatement("select * from student where id=?");
            stmt.setInt(1, id);
            // 执行查询并获取结果集
            ResultSet rs = stmt.executeQuery();
            // 遍历结果集，打印每个学生的详细信息
            if (rs.next()) {
                // 创建Student对象，并填充从数据库查询到的信息
                Student student = new Student();
                student.setId(id);
                student.setName(rs.getString("name"));
                student.setAge(rs.getInt("age"));
                student.setScore(rs.getFloat("score"));
                // 将查询到的学生信息存入Redis缓存
                jedis.rpush(key, student.getName(), String.valueOf(student.getAge()), String.valueOf(student.getScore()));
                return student;
            } else {
                System.out.println("mysql数据库中没有找到该学生");
            }
        } catch (SQLException e) {
            System.out.println(e);
        }
        return null;
    }
// 运行程序，输出结果如下：
从mysql中获取数据
id=1,name=Peter,age=18,score=100.0
从redis中获取数据
id=1,name=Peter,age=18,score=100.0

// 可以看出第一次缓存中没有数据，先从mysql中获取数据，然后将结果存入redis,下次直接从redis中取结果
```

#### 模拟缓存穿透现象
> 如果项目中引入了redis，那么多次请求相同数据且该数据存在于数据库的场景里能直接从Redis中读取数据，有效地降低对数据库的访问压力。在高并发的场景里，如果频繁地请求不存在于数据库的数据，就会引发缓存穿透的现象。
> 在上面的例子中，只存在id为1，2，3中的数据，如果请求id为4，5等不存在的数据，每次查询Redis缓存都不可能找到数据，所以都会继续向数据库发查询请求，高并发的查询请求会“穿透redis缓存”，集中到数据库上，这样就会给数据库造成很大的压力，严重时数据库甚至会崩溃，从而无法继续接受请求，造成严重的产线问题，这就是缓存穿透现象。

```java
        // 缓存穿透现象模拟，相当于高并发场景直接查询数据库
        for (int i = 1; i <= 10000; i++) { 
            int idNotExists=10;
            Student student = getStudent(idNotExists); // 从数据库中获取指定ID为10(不存在)的学生信息
            if (student != null) { 
                System.out.print("id=" + student.getId() + ",");
                System.out.print("name=" + student.getName() + ",");
                System.out.print("age=" + student.getAge() + ",");
                System.out.println("score=" + student.getScore()); // 打印学生的各项信息
            }
        }
// 运行程序，输出结果如下：
// 打印10000次，每次从数据库中获取指定ID为10(不存在)的学生信息，给数据库造成巨大压力
从mysql中获取数据
mysql数据库中没有找到该学生
...
从mysql中获取数据
mysql数据库中没有找到该学生
```

#### 模拟内存使用不当的场景
> 因为Redis是把数据存储到内存中的，所以从Redis缓存中读取数据的效率要高于Mysql数据库，如果每次在设置缓存时不设置超时时间，那么每次设置的缓存都会一直保存在内存中，久而久之，就会导致内存溢出问题。

```java
 /**
     * 演示Redis操作中的不当用法
     * 此方法展示在使用Redis进行大量数据操作时，不当的内存管理可能导致的潜在问题
     * 通过在循环中不断向Redis列表添加数据，可能导致JVM内存溢出
     */
    static void improperUse() {
        // 创建Redis客户端
        Jedis jedis = new Jedis("localhost", 6379);
        // 打印当前JVM空闲内存量，单位为M
        System.out.println((Runtime.getRuntime().freeMemory() >> 20) + "M");
        // 循环100000次，每次向Redis中添加数据
        for (int i = 0; i < 100000; i++) {
            // 创建一个包含随机字符串的键
            String key = "Stu:" + i;
            // 将键添加到Redis列表中
            jedis.rpush(key, String.valueOf(i));
        }
        // 再次打印JVM空闲内存量，单位为M
        System.out.println((Runtime.getRuntime().freeMemory() >> 20) + "M");
    }
// 运行程序，输出结果如下：
// 这里可以看到内存是不断减少的，如果数据量更大，或者时间更长，极有可能造成OOM，所以redis中的数据一定要设置过期时间
502M
499M
```

### 4.Redis缓存实战分析
#### 缓存不存在的键，以防穿透
```java
    static void avoidPenetrateDemo() {
        init();
        // 缓存穿透现象模拟
        for (int i = 1; i <= 3; i++) {
            int idNotExists = 10;
            Student student = getStudentV2(idNotExists); // 从数据库中获取指定ID为10(不存在)的学生信息
            if (student != null) {
                System.out.print("id=" + student.getId() + ",");
                System.out.print("name=" + student.getName() + ",");
                System.out.print("age=" + student.getAge() + ",");
                System.out.println("score=" + student.getScore()); // 打印学生的各项信息
            }
        }
    }

    static Student getStudentV2(int id) {
        // 构建Redis中的学生信息键名
        String key = "studentV2:" + id;
        // 检查Redis中是否存在该学生信息
        if (jedis.exists(key)) {
            System.out.println("从redis中获取数据");
            // 从Redis中获取学生信息，并映射到Student对象中
            List<String> list = jedis.lrange(key, 0, 2);
            Student student = new Student();
            student.setId(id);
            student.setName(list.get(0));
            student.setAge(Integer.parseInt(list.get(1)));
            student.setScore(Float.parseFloat(list.get(2)));
            return student;
        }
        System.out.println("从mysql中获取数据");
        try {
            // 准备SQL语句，用于查询学生信息
            PreparedStatement stmt = conn.prepareStatement("select * from student where id=?");
            stmt.setInt(1, id);
            // 执行查询并获取结果集
            ResultSet rs = stmt.executeQuery();
            // 遍历结果集，打印每个学生的详细信息
            if (rs.next()) {
                // 创建Student对象，并填充从数据库查询到的信息
                Student student = new Student();
                student.setId(id);
                student.setName(rs.getString("name"));
                student.setAge(rs.getInt("age"));
                student.setScore(rs.getFloat("score"));
                // 将查询到的学生信息存入Redis缓存
                jedis.rpush(key, student.getName(), String.valueOf(student.getAge()), String.valueOf(student.getScore()));
                return student;
            } else {
                // v2与上面的版本只有这里不同
                jedis.rpush(key, "", "0", "0");
                System.out.println("mysql数据库中没有找到该学生");
            }
        } catch (SQLException e) {
            System.out.println(e);
        }
        return null;
    }
// 运行程序，输出结果如下：
从mysql中获取数据
mysql数据库中没有找到该学生
从redis中获取数据
id=10,name=,age=0,score=0.0
从redis中获取数据
id=10,name=,age=0,score=0.0
// 可以看到，只有第一次去数据库查询，后面都从redis缓存中查询，减轻了数据库压力，大大提升了查询效率
```

#### 合理设置缓存超时时间，以防内存溢出
> 其他代码都一样，只是在保存到redis中之后，给key设置了缓存时间，`jedis.expire(key,60*5); // 缓存5分钟`,这样可以避免因对象永不过期而导致的内存溢出问题，对于缓存的时间合理性，需要根据业务需求合理设置，如果太长，依然会导致OOM，如果太短，缓存中的对象会过早失效，从而加重数据库的负担。
```java
  // 遍历结果集，打印每个学生的详细信息
            if (rs.next()) {
                // 创建Student对象，并填充从数据库查询到的信息
                Student student = new Student();
                student.setId(id);
                student.setName(rs.getString("name"));
                student.setAge(rs.getInt("age"));
                student.setScore(rs.getFloat("score"));
                // 将查询到的学生信息存入Redis缓存
                jedis.rpush(key, student.getName(), String.valueOf(student.getAge()), String.valueOf(student.getScore()));
                jedis.expire(key,60*5); // 缓存5分钟
                return student;
            } else {
                jedis.rpush(key, "", "0", "0");
                jedis.expire(key,60*30); // 缓存30分钟
                System.out.println("mysql数据库中没有找到该学生");
            }
```

#### 超时时间外加随机数，以防穿透，造成缓存雪崩
> 所有的key都设置为相同的过期时间，有个很大的问题就是，如果在某一时刻，批量加入的一千万个缓存数据同时过期，那么这批数据同时失效，从而对这批数据的请求都会发送到数据库，如果此时并发负载比较重，那么数据库同样会崩溃。解决方法是设置超时时间时加一个随机数。

```java
    /**
     * 演示Redis键的随机过期时间设置
     * 本方法旨在展示如何在Redis中设置键，并为每个键分配一个随机的过期时间
     * 它首先清除Redis的所有现有数据，然后设置5个键，每个键的过期时间都是一个随机数加上一个固定值
     * 之后，它通过休眠6秒来模拟超时，然后检查每个键是否仍然存在
     * 
     * @throws InterruptedException 如果线程在休眠期间被中断
     */
    static void randomExpireTimeDemo() throws InterruptedException {
        // 创建Jedis实例，连接到本地Redis服务器
        Jedis jedis = new Jedis("localhost", 6379);
        // 清除Redis中的所有数据，以便开始新的演示
        jedis.flushAll();
        // 定义一个固定的过期时间秒数
        int fixedTime = 5;
        // 循环5次，每次设置一个键和随机过期时间
        for (int i = 0; i < 5; i++) {
            // 生成一个0到2之间的随机数，作为随机过期时间部分
            int randomTime = (int) (Math.random() * 3);
            // 构造键名
            String key = "stu:" + i;
            // 设置键值对
            jedis.set(key, "value" + i);
            // 为键设置过期时间，是随机时间和固定时间之和
            jedis.expire(key, randomTime + fixedTime);
        }
        // 休眠6秒,模拟超时，以便观察键的过期效果
        Thread.sleep(6000);
        // 再次循环，检查并打印每个键是否存在
        for (int i = 0; i < 5; i++) {
            // 构造键名
            String key = "stu:" + i;
            // 检查键是否存在，并打印结果
            if (jedis.exists(key)) {
                System.out.println(key + " exists.");
            } else {
                System.out.println(key + " not exists.");
            }
        }
    }
// 运行程序，输出结果如下：
stu:0 exists.
stu:1 exists.
stu:2 not exists.
stu:3 exists.
stu:4 not exists.

// 可以看到，在常规超时时间后，并不是所有键同时失效，说明设置了随机过期时间，可以防止缓存穿透造成的雪崩。
```
