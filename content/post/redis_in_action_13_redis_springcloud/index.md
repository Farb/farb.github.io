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

SpringCloud Gateway：网关，用于实现服务间的路由，负载均衡，限流等。
SpringCloud Config：配置中心，用于管理配置文件，包括配置文件的版本控制，配置文件的发布等。
SpringCloud Bus：消息总线，用于实现服务间的通信，包括服务间的消息发布，服务间的消息订阅等。
SpringCloud Sleuth：分布式追踪，用于实现服务间的调用链路追踪，包括调用链路追踪的记录，调用链路追踪的日志等。
SpringCloud Circuit Breaker：断路器，用于实现服务间的容错，包括服务间的调用超时，服务间的调用失败等。
SpringCloud Security：安全中心，用于实现服务间的安全认证，包括服务间的用户认证，服务间的角色认证等。
SpringCloud LoadBalancer：负载均衡，用于实现服务间的负载均衡，包括服务间的轮询，随机等。
SpringCloud Discovery：服务发现，用于实现服务间的服务发现，包括服务间的服务注册，服务间的服务发现等。

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

### 包含Redis和Eureka的架构图

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
        <!--        <dependency>-->
        <!--            <groupId>org.springframework.boot</groupId>-->
        <!--            <artifactId>spring-boot-starter-data-jpa</artifactId>-->
        <!--            <version>RELEASE</version>-->
        <!--            <scope>compile</scope>-->
        <!--        </dependency>-->
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
    @GetMapping("hello")
    public String hello() {
        return "hello,this is Risk Service";
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
        return "Hello World,this is Order service";
    }

    @GetMapping("getRiskInfo")
    public String getRiskInfo() {
        // 一个服务可能在多个服务器节点上运行，所以一个服务可能对应多个实列
        List<ServiceInstance> serviceInstances = discoveryClient.getInstances("RiskService");
        if (!serviceInstances.isEmpty()) {
            return restTemplate.getForObject("http://RiskService/hello", String.class);
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

分别运行两个服务RiskService和OrderService，然后访问OrderService的接口http://localhost:2222/getRiskInfo，会自动调用RiskService的接口。

```bash
PS C:\Users\farbg> curl http://host.docker.internal:2222/getRiskInfo


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
# 返回的结果会在下面两种结果轮询
hello,this is Risk Service
hello,this is Risk Service,and this is node 2
hello,this is Risk Service
hello,this is Risk Service,and this is node 2
```