---
title: 基于Docker的Redis实战-Redis与SpringCloud微服务整合
description:
slug: redis_in_action_13_redis_springcloud
date: 2024-11-24
image: 
categories:
    - Redis
tags:
    - redis
    - docker
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---
源码链接： https://gitee.com/farb/architect-practicer-code 

## 微服务和SpringCloud相关概念

### 传统架构和微服务的比较

传统架构：所有模块都是集中开发，集中部署，集中运行的，耦合度比较高，模块之间通过代码直接调用，数据库和缓存也属于整个系统，包括所有模块的数据。

微服务：每个模块都是单独开发，单独部署，单独运行的，解耦度较高，模块之间通过接口调用，数据库和缓存也属于每个模块，每个模块的数据也只属于自己。整个系统的扩展能力较强，能用较小的代价扩展新的功能模块。

### 微服务和SpringCloud的相关概念
微服务是一种设计风格，有不同的实现方式，而SpringCould是一个微服务框架，它提供了一系列的解决方案，如服务注册中心、配置中心、路由、网关、负载均衡、断路器、监控、网关限流等。

 Gateway：网关，用于实现服务间的路由，负载均衡，限流等。

 Config：配置中心，用于管理配置文件，包括配置文件的版本控制，配置文件的发布等。

 Bus：消息总线，用于实现服务间的通信，包括服务间的消息发布，服务间的消息订阅等。

 Sleuth：分布式追踪，用于实现服务间的调用链路追踪，包括调用链路追踪的记录，调用链路追踪的日志等。

 Circuit Breaker：断路器，用于实现服务间的容错，包括服务间的调用超时，服务间的调用失败等。

 Security：安全中心，用于实现服务间的安全认证，包括服务间的用户认证，服务间的角色认证等。

 LoadBalancer：负载均衡，用于实现服务间的负载均衡，包括服务间的轮询，随机等。

 Discovery：服务发现，用于实现服务间的服务发现，包括服务间的服务注册，服务间的服务发现等。

SpringCloud的常用组件如下：

|组件|功能|
|--|--|
|JPA|ORM框架|
|Eureka|服务治理组件，通过该组件，系统可以动态地实现微服务组件治理|
|Hystrix|容错保护组件，能提供自动熔断和限流等方面的功能|
|Ribbon|负载均衡组件，能够把请求均衡地分摊到各服务组件上，包括轮询、随机、一致性哈希等|
|Zuul|网关组件，能够实现服务间的路由|
|Spring Cloud Config|配置中心组件，能够实现配置文件的管理|


## 多模块整合Redis，构建微服务体系

### 用Docker准备Redis和MySQL集群环境

创建一个一主一从的Redis集群环境:

``` bash
PS C:\Users\farbg> docker run -itd --name master-redis -p 6379:6379 redis
6c0478e385052a2bb2547e3436e236258cd3c7b75c4af234cede446b9987b39f
PS C:\Users\farbg> docker inspect -f '{{.NetworkSettings.IPAddress}}' master-redis
172.17.0.4

PS C:\Users\farbg> docker run -itd --name slave-redis -p 6380:6380 redis
81441fe049dda8b567bfef6235f7a9bc760cb383cbd4a77c0461ee164dcc28d7
PS C:\Users\farbg> docker inspect -f '{{.NetworkSettings.IPAddress}}' slave-redis
172.17.0.2

PS C:\Users\farbg> docker exec -it slave-redis bash
root@4cc2ca237b09:/data# redis-cli
127.0.0.1:6379> SLAVEOF 172.17.0.4 6379
OK
127.0.0.1:6379> info replication
# Replication
role:slave
master_host:172.17.0.4
master_port:6379
master_link_status:up
master_last_io_seconds_ago:1
master_sync_in_progress:0
slave_read_repl_offset:14
slave_repl_offset:14
slave_priority:100
slave_read_only:1
replica_announced:1
connected_slaves:0
master_failover_state:no-failover
master_replid:cc12d98a9045805969db2fa546a002bb35234c53
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:14
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:14

root@6c0478e38505:/data# redis-cli
127.0.0.1:6379> info replication
# Replication
role:master
connected_slaves:1
slave0:ip=172.17.0.2,port=6379,state=online,offset=112,lag=1
master_failover_state:no-failover
master_replid:cc12d98a9045805969db2fa546a002bb35234c53
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:112
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:112

# 验证主从模式是否正常工作
# 在主节点set
127.0.0.1:6379> set hello world
OK

# 在从节点get
127.0.0.1:6379> get hello
"world"
```

搭建一个一主一从的MySQL集群环境：

```bash
# 本机中新建mysql的配置文件 D:\ArchitectPracticer\Redis\MySql\MasterMySql\conf\my.cnf ，内容如下：
[mysqld]
pid-file=/var/run/mysqld/mysqld.pid
socket=/var/run/mysqld/mysqld.sock
datadir=/var/lib/mysql
server-id=1 # 主服务器的id，这个id不能重复
log-bin=mysql-master-bin

# 启动主服务器，设置环境变量，MYSQL_ROOT_PASSWORD为root用户的密码，同时挂载配置文件目录和数据目录
PS D:\code\blogs\farb.github.io> docker run -itd --name master-mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 -v D:\ArchitectPracticer\Redis\MySql\MasterMySql\conf:/etc/mysql/conf.d -v D:\ArchitectPracticer\Redis\MySql\MasterMySql\data:/var/lib/mysql mysql
05e2c449e667b7fbd6898f9e84306440d1afff5c7e75624a438af082157ca75d

# 查看运行的容器
PS D:\code\blogs\farb.github.io> docker ps
CONTAINER ID   IMAGE     COMMAND                   CREATED              STATUS              PORTS                               NAMES
7b1067615f0d   mysql     "docker-entrypoint.s…"   About a minute ago   Up About a minute   0.0.0.0:3306->3306/tcp, 33060/tcp   master-mysql

# 获取主服务器的IP地址
D:\code\blogs\farb.github.io> docker inspect master-mysql -f "{{.NetworkSettings.IPAddress}}" 
172.17.0.3

# 登录主服务器
PS D:\code\blogs\farb.github.io> docker exec -it master-mysql bash
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

mysql> SHOW BINARY LOG STATUS;
+-------------------------+----------+--------------+------------------+-------------------+
| File                    | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+-------------------------+----------+--------------+------------------+-------------------+
| mysql-master-bin.000007 |      158 |              |                  |                   |
+-------------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)

```

创建从数据库：

```bash
# mysql从服务器的配置文件如下
[mysqld]
pid-file=/var/run/mysqld/mysqld.pid
socket=/var/run/mysqld/mysqld.sock
datadir=/var/lib/mysql
server-id=2 # 从服务器的id，这个id不能和主服务器的一致
log-bin=mysql-slave-bin # 二进制文件的名字

# 启动从服务器
PS D:\code\blogs\farb.github.io> docker run -itd --name slave-mysql -p 3316:3306 -e MYSQL_ROOT_PASSWORD=123456 -v D:\ArchitectPracticer\Redis\MySql\SlaveMySql\conf:/etc/mysql/conf.d -v D:\ArchitectPracticer\Redis\MySql\SlaveMySql\data:/var/lib/mysql mysql 

PS D:\code\blogs\farb.github.io> docker exec -it slave-mysql /bin/bash

# 确认可以在从服务器的容器中连接到主服务器
bash-5.1# mysql -h 172.17.0.3 -u root -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 18
Server version: 8.4.0 MySQL Community Server - GPL

Copyright (c) 2000, 2024, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

#  exit退出主服务器的连接
mysql> exit
Bye

# 连接到从服务器
bash-5.1# mysql -h localhost -u root -p
mysql: [Warning] World-writable config file '/etc/mysql/conf.d/my.cnf' is ignored.
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 11
Server version: 8.4.0 MySQL Community Server - GPL

Copyright (c) 2000, 2024, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

# 配置主从关系，这里注意`change master to`已经在MySql8中被`CHANGE REPLICATION SOURCE TO`取代。每个字段的意思很明显，主要是`SOURCE_log_pos`和`SOURCE_log_file`要和上面`show binary log status;`的结果一致。`
mysql> CHANGE REPLICATION SOURCE TO SOURCE_host='172.17.0.3',SOURCE_port=3306,SOURCE_user='root',SOURCE_password='123456',SOURCE_log_pos=158,SOURCE_log_file='mysql-master-bin.000007';
Query OK, 0 rows affected, 2 warnings (0.04 sec)

# start slave;也变成了start replica; 详见 https://dev.mysql.com/doc/refman/8.4/en/start-replica.html
mysql> start replica;
Query OK, 0 rows affected (0.01 sec)


# 查看主从复制状态，命令结尾符\G是以垂直列表的形式显示。看到Replica_IO_Running: Yes 和 Replica_SQL_Running: Yes，说明配置主从复制成功
mysql> show replica status\G;
*************************** 1. row ***************************
             Replica_IO_State: Waiting for source to send event
                  Source_Host: 172.17.0.2
                  Source_User: root
                  Source_Port: 3306
                Connect_Retry: 60
              Source_Log_File: mysql-master-bin.000002
          Read_Source_Log_Pos: 158
               Relay_Log_File: 0f827dad7de8-relay-bin.000003
                Relay_Log_Pos: 335
        Relay_Source_Log_File: mysql-master-bin.000002
           Replica_IO_Running: Yes
          Replica_SQL_Running: Yes
... 省略一部分输出
1 row in set (0.00 sec)

ERROR:
No query specified


# 回到主服务器，创建数据库
mysql> create database  RiskDb;
Query OK, 1 row affected (0.01 sec)

mysql> use RiskDb;
Database changed
mysql> create table riskinfo (id int not null primary key,level varchar(20),userId varchar(20));
Query OK, 0 rows affected (0.02 sec)

# 表示用户id为003的用户有高危风险,不能下单
mysql> insert into riskinfo (id,level,userid) values(1,'High','003');
Query OK, 1 row affected (0.01 sec)

# 可以看到主服务器中有新建的数据库 RiskDb
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| MasterSlaveDemo    |
| mysql              |
| performance_schema |
| redisDemo          |
| RiskDb             |
| sys                |
+--------------------+
7 rows in set (0.00 sec)

# 再去从服务器中查看，检查是否同步成功，结果发现同步成功
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| MasterSlaveDemo    |
| mysql              |
| performance_schema |
| redisDemo          |
| RiskDb             |
| sys                |
+--------------------+
7 rows in set (0.00 sec)

```

至此，一主一从的Redis和MySQL主从复制搭建完成。

|Docker容器名|IP和端口|说明|
|:---------:|:------:|:--:|
| master-mysql|172.17.0.3:3306|MySQL主服务器|
| slave-mysql|172.17.0.5:3316|MySQL从服务器|
| master-redis|172.17.0.4:6379|Redis主服务器|
| slave-redis|172.17.0.2:6380|Redis从服务器|

### 包含Redis和Eureka的架构图

[![pA51WDA.png](https://s21.ax1x.com/2024/11/28/pA51WDA.png)](https://imgse.com/i/pA51WDA)

### 开发Eureka服务端

新建一个SpringBoot项目，添加依赖，pom.xml如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://maven.apache.org/POM/4.0.0"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.3.2</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>farb.top</groupId>
    <artifactId>EurekaServer</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>EurekaServer</name>
    <description>EurekaServer</description>

    <properties>
        <java.version>17</java.version>
        <spring-cloud.version>2023.0.2</spring-cloud.version>
    </properties>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

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

EurekaServerApplication.java代码如下：

```java
@EnableEurekaServer
@SpringBootApplication
public class EurekaServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }

}
```

application.yml配置如下：

```yml
server:
  port: 8888
eureka:
  instance:
    hostname: localhost
  client:
    fetch-registry: false
    register-with-eureka: false 
    service-url:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka
```

运行EurekaServerApplication，访问http://localhost:8888/，可以看到如下界面，说明Eureka服务端已经启动成功。

[![pAhKHO0.png](https://s21.ax1x.com/2024/11/24/pAhKHO0.png)](https://imgse.com/i/pAhKHO0)

### 开发Eureka客户端

新建一个SpringBoot项目EurekaClientRiskApp，作为风险模块，添加依赖，pom.xml如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://maven.apache.org/POM/4.0.0"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.3.2</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>farb.top</groupId>
    <artifactId>EurekaClientRiskApp</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>EurekaClientRiskApp</name>
    <description>EurekaClientRiskApp</description>
    <properties>
        <java.version>17</java.version>
        <spring-cloud.version>2023.0.2</spring-cloud.version>
    </properties>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
            <version>3.3.2</version>
            <scope>compile</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
            <version>RELEASE</version>
            <scope>compile</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
            <version>3.4.0</version>
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>8.0.33</version>
        </dependency>
        <dependency>
            <groupId>com.alibaba.fastjson2</groupId>
            <artifactId>fastjson2</artifactId>
            <version>2.0.43</version>
        </dependency>
    </dependencies>
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

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

下面是具体代码：

```java
@EnableDiscoveryClient
@SpringBootApplication
public class EurekaClientRiskAppApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaClientRiskAppApplication.class, args);
    }
}

@RestController
public class RiskController {
    private final RiskService riskService;
    @Value("${server.port}")
    private String port;

    public RiskController(RiskService riskService) {
        this.riskService = riskService;
    }

    @GetMapping("hello")
    public String hello() {
        return "hello,this is Risk Service";
    }

    @GetMapping("getRiskLevel/{userId}")
    public String getRiskLevel(@PathVariable String userId) {
        System.out.println("当前端口号为：" + this.port);
        return riskService.getRiskLevel(userId);
    }  
}

@Entity
@Table(name = "riskinfo")
@Data
@AllArgsConstructor
@NoArgsConstructor
public class Risk {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private int id;
    private String level;

    private String userid;
}

@Repository
public class RedisRiskRepo {
    @Autowired
    private RedisTemplate<String, String> redisTemplate;

    public void saveRiskLevel(String key, int expireTime, Risk risk) {
        redisTemplate.opsForValue().set(key, JSON.toJSONString(risk), expireTime, TimeUnit.MINUTES);
    }

    public Risk getRiskLevel(String key) {
        String json = redisTemplate.opsForValue().get(key);
        return JSON.parseObject(json, Risk.class);
    }
}

@Repository
public interface RiskRepo extends JpaRepository<Risk, Long> {
    Risk getRiskByUserid(String userid);
}

@Service
public class RiskService {
    private static final String RISK_KEY_PREFIX = "risk_";
    @Autowired
    private RiskRepo riskRepo;
    @Autowired
    private RedisRiskRepo redisRiskRepo;

    public String getRiskLevel(String userId) {
        Risk risk = redisRiskRepo.getRiskLevel(RISK_KEY_PREFIX + userId);
        if (risk != null) {
            System.out.println("从redis获取数据");
            return risk.getLevel();
        }
        risk = riskRepo.getRiskByUserid(userId);
        System.out.println("从mysql获取数据");
        if (risk != null) {
            redisRiskRepo.saveRiskLevel(RISK_KEY_PREFIX + userId, 60, risk);
            return risk.getLevel();
        }
        return "";
    }
}
```

```yml
server:
  port: 1111
spring:
  application:
    name: RiskService
  data:
    redis:
      host: localhost
      port: 6379
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8888/eureka
    fetch-registry: true
    register-with-eureka: true
```

复制一个项目EurekaClientRiskApp2，只修改yml中的端口号为1222，用来区分负载均衡生效。

新建一个SpringBoot项目EurekaClientOrderApp，作为订单模块，添加依赖，pom.xml如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://maven.apache.org/POM/4.0.0"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.3.2</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>farb.top</groupId>
    <artifactId>EurekaClientOrderApp</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>EurekaClientOrderApp</name>
    <description>EurekaClientOrderApp</description>
    <properties>
        <java.version>17</java.version>
        <spring-cloud.version>2023.0.2</spring-cloud.version>
    </properties>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
            <version>3.4.0</version>
        </dependency>
    </dependencies>
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

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

```java
@SpringBootApplication
@EnableDiscoveryClient
public class EurekaClientOrderAppApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaClientOrderAppApplication.class, args);
    }

}

@RestController
public class HelloController {
    @LoadBalanced
    @Resource
    private RestTemplate restTemplate;

    @Resource
    private DiscoveryClient discoveryClient;

    @GetMapping("hello")
    public String hello() {
        // 一个服务可能在多个服务器节点上运行，所以一个服务可能对应多个实列
        List<ServiceInstance> serviceInstances = discoveryClient.getInstances("RiskService");
        if (!serviceInstances.isEmpty()) {
            return restTemplate.getForObject("http://RiskService/hello", String.class);
        }
        return "RiskService not found";
    }

    @GetMapping("getRiskInfo")
    public String getRiskInfo() {
        // 一个服务可能在多个服务器节点上运行，所以一个服务可能对应多个实列
        List<ServiceInstance> serviceInstances = discoveryClient.getInstances("RiskService");
        if (!serviceInstances.isEmpty()) {
            return restTemplate.getForObject("http://RiskService/getRiskLevel/003", String.class);
        }
        return "RiskService not found";
    }
}

@Configuration
public class AppConfig {
    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

```yml
spring:
  application:
    name: OrderService
eureka:
  client:
    fetch-registry: true
    register-with-eureka: true
    service-url:
      defaultZone: http://localhost:8888/eureka
server:
  port: 2222

```

分别运行两个服务RiskService和OrderService，然后访问OrderService的接口http://localhost:2222/hello，会自动调用RiskService的接口。

```bash
PS C:\Users\farbg> curl http://host.docker.internal:2222/hello


StatusCode        : 200
StatusDescription :
Content           : hello,this is Risk Service
RawContent        : HTTP/1.1 200
                    Content-Length: 26
                    Content-Type: text/plain;charset=UTF-8
                    Date: Mon, 25 Nov 2024 15:19:23 GMT

                    hello,this is Risk Service
Forms             : {}
Headers           : {[Content-Length, 26], [Content-Type, text/plain;charset=UTF-8], [Date, Mon, 25 Nov 2024 15:19:23 G
                    MT]}
Images            : {}
InputFields       : {}
Links             : {}
ParsedHtml        : mshtml.HTMLDocumentClass
RawContentLength  : 26

# 返回的结果会在下面两种结果轮询
hello,this is Risk Service
hello,this is Risk Service,and this is node 2
hello,this is Risk Service
hello,this is Risk Service,and this is node 2
```

## Redis与Ribbon集成

复制一个RiskService模块，将配置文件的端口修改如下，其他都不变：

``` yml
server:
  port: 1222
spring:
  application:
    name: RiskService
  data:
    redis:
      host: localhost
      port: 6379
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8888/eureka
    fetch-registry: true
    register-with-eureka: true
```

然后再修改一下控制器，目的是能区分出响应是从哪个节点返回的：

```java
@RestController
public class RiskController {
    @GetMapping("hello")
    public String hello() {
        return "hello,this is Risk Service,and this is node 2";
    }
}
```

启动两个服务，然后查看Eureka服务器的实例列表如下：

[![pAhfzZR.png](https://s21.ax1x.com/2024/11/25/pAhfzZR.png)](https://imgse.com/i/pAhfzZR)

然后访问http://host.docker.internal:2222/getRiskInfo，会自动调用RiskService的接口，因为Ribbon默认是轮询的方式，所以每次访问结果都不一样。

``` bash
# 返回的结果会在两个服务节点切换，而且一开始是从mysql获取数据，以后从redis获取数据

当前端口号为：1222
从mysql获取数据
当前端口号为：1222
从redis获取数据
当前端口号为：1222
从redis获取数据
当前端口号为：1111
从redis获取数据
当前端口号为：1111
从redis获取数据
```