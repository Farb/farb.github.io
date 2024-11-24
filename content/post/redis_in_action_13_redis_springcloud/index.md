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
            <version>4.1.3</version>
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
