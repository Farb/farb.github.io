---
title: SpringCloud Alibaba 入门实战
description:
slug: springcloudalibaba_01_introduction
date: 2024-12-07
image: 
categories:
    - SpringCloud
    - SpringCloudAlibaba
tags:
    - SpringCloudAlibaba
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---

源码链接： https://gitee.com/farb/architect-practicer-code 

## 单体架构、SOA与微服务

### 单体架构
一个系统的所有模块在一个工程中开发，部署时将所有代码打包。

优点：业务初期，功能简单，开发成本低，效率高。

缺点：随着业务量不断增大，会出现如下问题：

1. 代码复杂度高：所有代码杂糅在一起，依赖紧密，修改一处可能导致其他地方出现问题。
2. 体积大，部署慢：假设一个完整的包是500M，如果只改了一行代码，也需要重新部署整个包。
3. 程序出错影响服务稳定性：某个功能的bug可能导致整个进程的崩溃，影响服务可用性。
4. 阻碍技术的更新：一般来说单体架构只能使用同一个技术栈，比如纯java或纯c#。
5. 集群扩容不合理：因为单体架构所有模块一起部署，假如只是用户模块需要扩容，但是其他模块也跟着扩容，造成资源浪费，而且扩容时间较长。

### SOA （Service Oriented Architecture）
微服务是SOA体系结构不断演进的结果。SOA和微服务本质上都在拆分服务，为了降低耦合，提升代码的复用性。但微服务更突出服务的独立性，是更细粒度地划分服务。拆分粒度没有统一标准，需要自己权衡利弊。

### 微服务

微服务是一种架构风格，它把一个复杂的系统拆分成多个小的服务，每个服务只做一件事情，服务之间互相调用。

优点：

1. 代码复杂度低：根据业务细粒度拆分成多个小服务，业务功能清晰，代码体积小，便于理解维护。
2. 技术选型不受限：单个服务可以根据自身业务选择合适的技术栈。
3. 独立部署：如果修改一个后台服务，修改完毕后只需要部署这一个服务即可，其他服务不动，影响范围小，也减少了测试和部署的时间。
4. 服务的可伸缩性：根据业务发展，哪个服务承载量到达了上限，就多部署一些节点，反之，哪些业务量不大，就减少一些节点。
5. 错误隔离：当某个服务出现故障时，不会影响其他服务，不会影响整个系统。比如后台管理服务出现故障，不会影响用户服务。
6. 容易分库：每个服务独立部署，也可以独立连接一个数据库，省去单体架构的分库成本。

缺点：

万事万物都是一分为二的，有好必然有坏。

1. 构建微服务系统复杂：构建微服务系统需要很多技术，比如注册中心、配置中心、服务调用、服务网关、消息总线、监控中心、分布式事务等。需要考虑网络延迟和网络故障的影响。
2. 服务依赖：假如三个服务A、B、C,A调用B,B调用C，如果C不得已必须修改，可能B要修改，A可能也因为B的修改也要修改。
3. 数据一致性问题：各个服务都是独立的进程，传统的数据库事务失效，假设A调用B，恰好遇到网络延迟或网络抖动等网络故障，A服务数据回滚，而B数据可能已经修改保存。
4. 排除故障困难：微服务调用链较长，可能涉及跨多个服务，因此要定位线上问题，不得不同时查看多个服务问题。
5. 运维和部署：要检查和监控多个服务的健康状态，快速部署多个服务，以及根据服务的负载情况进行动态伸缩，都是不小的挑战。

## 微服务所需要的技术栈
1. 服务注册与发现：当调用接口服务时，首先从服务注册中心获取被调用服务的真实地址，然后调用具体某个服务，调用方不需要知道真实地址。这样很容易实现目标服务的负载均衡，服务之间也实现了解耦。
2. 服务调用：调用方真正调用目标服务的技术（HTTP,RPC）
3. 服务熔断、限流、降级：保证服务的稳定性。
4. 负载均衡：决定调用服务集群中的哪个服务。
5. 配置中心：集中式管理项目的配置，方便修改，解决重复配置，实时生效。
6. 消息队列：服务之间可以异步、解耦，减少调用链路。
7. 服务网关：可以用来做认证，灰度发布等。
8. 服务监控：监控服务的健康状态
9. 消息总线：事件传播与通知，解耦微服务依赖关系，数据一致性维护，异步通信等
10. 分布式链路追踪：查看接口调用链路，排除故障
11. 自动化构建部署：CI/CD

## Spring Cloud Alibaba是什么？
引用官网的一段话：https://sca.aliyun.com/docs/2023/overview/what-is-sca

Spring Cloud Alibaba 致力于提供微服务开发的一站式解决方案。此项目包含开发分布式应用服务的必需组件，方便开发者通过 Spring Cloud 编程模型轻松使用这些组件来开发分布式应用服务。

依托 Spring Cloud Alibaba，您只需要添加一些注解和少量配置，就可以将 Spring Cloud 应用接入阿里分布式应用解决方案，通过阿里中间件来迅速搭建分布式应用系统。

Spring Cloud Alibaba 提供的主要组件包括：

1. Nacos：服务注册中心、配置中心、服务元数据存储中心。
2. Sentinel：流量控制、熔断降级、系统负载保护。
3. Seata：分布式事务解决方案。
4. dubbo：远程调用RPC框架。
5. RocketMQ：阿里的开源分布式消息中间件，高性能、高可用、高吞吐量的金融级消息中间件。
6. OSS(收费): 文件存储服务
7. ScheduleX(收费)：分布式任务调度。
8. SMS(收费): 短信服务。

## Spring Cloud 微服务体系
Spring Cloud 是分布式微服务架构的一站式解决方案，它提供了一套简单易用的编程模型，使我们能在 Spring Boot 的基础上轻松地实现微服务系统的构建。 Spring Cloud 提供以微服务为核心的分布式系统构建标准。

Spring Cloud 本身并不是一个开箱即用的框架，它是一套微服务规范，共有两代实现。

1. Spring Cloud Netflix 是 Spring Cloud 的第一代实现，主要由 Eureka、Ribbon、Feign、Hystrix 等组件组成。
2. Spring Cloud Alibaba 是 Spring Cloud 的第二代实现，主要由 Nacos、Sentinel、Seata 等组件组成。

[![pA78O91.png](https://s21.ax1x.com/2024/12/07/pA78O91.png)](https://imgse.com/i/pA78O91)

## Spring Cloud Alibaba 定位

[![pA7Gi4A.png](https://s21.ax1x.com/2024/12/07/pA7Gi4A.png)](https://imgse.com/i/pA7Gi4A)

Spring Cloud Alibaba 是阿里巴巴结合自身丰富的微服务实践而推出的微服务开发的一站式解决方案，是 Spring Cloud 第二代实现的主要组成部分。吸收了 Spring Cloud Netflix 微服务框架的核心架构思想，并进行了高性能改进。自 Spring Cloud Netflix 进入停更维护后，Spring Cloud Alibaba 逐渐代替它成为主流的微服务框架。

同时 Spring Cloud Alibaba 也是国内首个进入 Spring 社区的开源项目。2018 年 7 月，Spring Cloud Alibaba 正式开源，并进入 Spring Cloud 孵化器中孵化；2019 年 7 月，Spring Cloud 官方宣布 Spring Cloud Alibaba 毕业，并将仓库迁移到 Alibaba Github OSS 下。

[![pA74NUe.png](https://s21.ax1x.com/2024/12/08/pA74NUe.png)](https://imgse.com/i/pA74NUe)

## 关于Spring Cloud Alibaba各个组件的版本兼容问题

https://sca.aliyun.com/docs/2023/overview/version-explain

[![pA7GA3t.png](https://s21.ax1x.com/2024/12/07/pA7GA3t.png)](https://imgse.com/i/pA7GA3t)

## Spring Cloud Alibaba 快速开始

### 安装运行Nacos服务

根据官网文档，下载压缩包或者直接使用docker镜像都可以。 https://nacos.io/download/nacos-server/

然后以standalone的方式运行Nacos服务，我这里以官方bat脚本运行，传入参数standalone，运行成功如下所示：

```bash

D:\ArchitectPracticer\MicroServiceTools\nacos-server-2.4.3\nacos\bin>startup.cmd -m standalone
"nacos is starting with standalone"

         ,--.
       ,--.'|
   ,--,:  : |                                           Nacos 2.4.3
,`--.'`|  ' :                       ,---.               Running in stand alone mode, All function modules
|   :  :  | |                      '   ,'\   .--.--.    Port: 8848
:   |   \ | :  ,--.--.     ,---.  /   /   | /  /    '   Pid: 22544
|   : '  '; | /       \   /     \.   ; ,. :|  :  /`./   Console: http://192.168.31.227:8848/nacos/index.html
'   ' ;.    ;.--.  .-. | /    / ''   | |: :|  :  ;_
|   | | \   | \__\/: . ..    ' / '   | .; : \  \    `.      https://nacos.io
'   : |  ; .' ," .--.; |'   ; :__|   :    |  `----.   \
|   | '`--'  /  /  ,.  |'   | '.'|\   \  /  /  /`--'  /
'   : |     ;  :   .'   \   :    : `----'  '--'.     /
;   |.'     |  ,     .-./\   \  /            `--'---'
'---'        `--`---'     `----'

2024-12-06 21:27:26,565 INFO Tomcat initialized with port(s): 8848 (http)

2024-12-06 21:27:26,977 INFO Root WebApplicationContext: initialization completed in 3424 ms

2024-12-06 21:27:31,041 INFO Will secure any request with [org.springframework.security.web.session.DisableEncodeUrlFilter@4cad79bc, org.springframework.security.web.context.request.async.WebAsyncManagerIntegrationFilter@2c63762b, org.springframework.security.web.context.SecurityContextPersistenceFilter@726934e2, org.springframework.security.web.header.HeaderWriterFilter@1163a27, org.springframework.security.web.csrf.CsrfFilter@3b55dd15, org.springframework.security.web.authentication.logout.LogoutFilter@3f6bf8aa, org.springframework.security.web.savedrequest.RequestCacheAwareFilter@e280403, org.springframework.security.web.servletapi.SecurityContextHolderAwareRequestFilter@42f85fa4, org.springframework.security.web.authentication.AnonymousAuthenticationFilter@7a9eccc4, org.springframework.security.web.session.SessionManagementFilter@92d1782, org.springframework.security.web.access.ExceptionTranslationFilter@28a9494b]

2024-12-06 21:27:31,288 INFO Adding welcome page: class path resource [static/index.html]

2024-12-06 21:27:31,783 INFO Exposing 1 endpoint(s) beneath base path '/actuator'

2024-12-06 21:27:31,789 WARN You are asking Spring Security to ignore Ant [pattern='/**']. This is not recommended -- please use permitAll via HttpSecurity#authorizeHttpRequests instead.

2024-12-06 21:27:31,805 INFO Will not secure Ant [pattern='/**']

2024-12-06 21:27:31,805 WARN You are asking Spring Security to ignore Mvc [pattern='/prometheus']. This is not recommended -- please use permitAll via HttpSecurity#authorizeHttpRequests instead.

2024-12-06 21:27:31,805 INFO Will not secure Mvc [pattern='/prometheus']

2024-12-06 21:27:31,805 WARN You are asking Spring Security to ignore Mvc [pattern='/prometheus/namespaceId/{namespaceId}']. This is not recommended -- please use permitAll via HttpSecurity#authorizeHttpRequests instead.

2024-12-06 21:27:31,805 INFO Will not secure Mvc [pattern='/prometheus/namespaceId/{namespaceId}']

2024-12-06 21:27:31,805 WARN You are asking Spring Security to ignore Mvc [pattern='/prometheus/namespaceId/{namespaceId}/service/{service}']. This is not recommended -- please use permitAll via HttpSecurity#authorizeHttpRequests instead.

2024-12-06 21:27:31,805 INFO Will not secure Mvc [pattern='/prometheus/namespaceId/{namespaceId}/service/{service}']

2024-12-06 21:27:31,867 INFO Tomcat started on port(s): 8848 (http) with context path '/nacos'

2024-12-06 21:27:31,867 INFO No TaskScheduler/ScheduledExecutorService bean found for scheduled processing

2024-12-06 21:27:31,901 INFO Nacos started successfully in stand alone mode. use embedded storage

2024-12-06 21:27:42,783 INFO Initializing Servlet 'dispatcherServlet'

2024-12-06 21:27:42,785 INFO Completed initialization in 1 ms
```

然后可以浏览器中打开 http://192.168.31.227:8848/nacos或localhost:8848/nacos 查看Nacos管理界面是否可以查看。

[![pA70w5T.png](https://s21.ax1x.com/2024/12/07/pA70w5T.png)](https://imgse.com/i/pA70w5T)

### 创建微服务

接下来就是创建微服务，启动，查看服务是否注册。

我这里使用的是当前最新的SpringCloudAlibaba依赖版本2023.0.3.2，支持的springboot版本是3.2.4。

创建一个父模块SpringCloudAlibabaPractice，只管理公共pom配置，如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>top.farb</groupId>
    <artifactId>SpringCloudAlibabaPractice</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>SpringCloudAlibabaPractice</name>
    <description>SpringCloudAlibabaPractice</description>
    <packaging>pom</packaging>
    <properties>
        <java.version>17</java.version>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <spring-boot.version>3.2.4</spring-boot.version>
        <spring-cloud-alibaba.version>2023.0.3.2</spring-cloud-alibaba.version>
    </properties>
    <!-- 后面所有演示模块都会以子模块的形式加入-->
    <modules>
        <module>FirstServiceDemo</module>
    </modules>
    <!-- 这里加一下子模块基本都会使用的依赖，避免子模块重复添加-->
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
    </dependencies>
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>${spring-boot.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <dependency>
                <groupId>com.alibaba.cloud</groupId>
                <artifactId>spring-cloud-alibaba-dependencies</artifactId>
                <version>2023.0.3.2</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.8.1</version>
                <configuration>
                    <source>17</source>
                    <target>17</target>
                    <encoding>UTF-8</encoding>
                </configuration>
            </plugin>
        </plugins>
    </build>

</project>

```

然后，创建一个子模块，例如FirstServiceDemo，只有一个Springboot启动类就行，再加一个application.yml如下：

```yml
server:
  port: 8080
spring:
  application:
    name: first-service
  cloud:
    nacos:
      discovery:
        server-addr: localhost:8848 # nacos注册中心地址
        namespace: # 如果为空，则使用public命名空间，否则需要配置nacos的命名空间id
```

启动应用，可以到nacos控制台页面看到服务已经注册成功。

[![pA70w5T.png](https://s21.ax1x.com/2024/12/07/pA70w5T.png)](https://imgse.com/i/pA70w5T)

至此，基本的框架就搭建好了。后面的实战会以这个作为基础。