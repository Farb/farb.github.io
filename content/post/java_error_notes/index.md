---
title: Java代码报错记录集
description:
slug: java_error_notes
date: 2024-08-14
image: 
categories:
    - java
tags:
    - java
    - error_notes
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---

### 1. getInputStream() must not be called against a directory: file://D:/DevTools/java/apache-maven-3.9.5/conf
> 这个错误是在IDEA中配置Maven的时候出现的，在配置Maven的setting文件路径的时候，如果配置的是一个conf目录或其他目录，就会报错，正确做法是指定setting.xml的全路径。

### 2. SLF4J: Failed to load class "org.slf4j.impl.StaticLoggerBinder
通常意味着 Simple Logging Facade for Java (SLF4J) 在启动时未能找到一个合适的日志实现。这可能是由于以下几个原因造成的：
1. 缺少日志实现库：项目中没有包含任何 SLF4J 的绑定实现，如 Logback 或 Log4j。
2. 多个日志实现库冲突：项目中包含了多个 SLF4J 绑定实现，导致 SLF4J 不确定使用哪一个。
3. 错误的日志实现库版本：项目中包含的日志实现库与 SLF4J 版本不兼容。**

``` xml
  <dependencies>
      <!-- SLF4J API -->
      <dependency>
          <groupId>org.slf4j</groupId>
          <artifactId>slf4j-api</artifactId>
          <version>1.7.36</version>
      </dependency>
      
      <!-- Logback as the logging implementation -->
      <dependency>
          <groupId>ch.qos.logback</groupId>
          <artifactId>logback-classic</artifactId>
          <version>1.2.11</version>
      </dependency>
  </dependencies>
  
```

### 3. Loading class `com.mysql.jdbc.Driver'. This is deprecated. The new driver class is `com.mysql.cj.jdbc.Driver'. The driver is automatically registered via the SPI and manual loading of the driver class is generally unnecessary.
**当你看到这条警告信息时，说明你在使用 MySQL 的 JDBC 驱动时正在加载旧版的驱动类 com.mysql.jdbc.Driver，而新版的驱动类应该是 com.mysql.cj.jdbc.Driver。这通常发生在使用 MySQL 8.0 及更高版本时。**

```java
// 不要这样写
Class.forName("com.mysql.jdbc.Driver");

// 按照提示，驱动会自动注册，所以不写也没问题
// 如果你需要显式指定驱动类，使用新版的驱动类
Class.forName("com.mysql.cj.jdbc.Driver");
```

### 4.Caused by: jakarta.persistence.PersistenceException: [PersistenceUnit: default] Unable to build Hibernate SessionFactory; nested exception is org.hibernate.exception.JDBCConnectionException: Unable to open JDBC Connection for DDL execution [Public Key Retrieval is not allowed] [n/a]

数据库连接字符串中加上：allowPublicKeyRetrieval=true

``` yml
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://localhost:3306/stockDB?allowPublicKeyRetrieval=true&serverTimezone=Asia/Shanghai&useUnicode=true&characterEncoding=utf-8&useSSL=false
    username: root
    password: 123456
```

### 5. Caused by: org.hibernate.service.spi.ServiceException: Unable to create requested service [org.hibernate.engine.jdbc.env.spi.JdbcEnvironment] due to: Unable to determine Dialect without JDBC metadata (please set 'jakarta.persistence.jdbc.url' for common cases or 'hibernate.dialect' when a custom Dialect implementation must be provided)

jpa是在spring节点下配置，而不是在spring.datasource下配置。

```yml
spring:
  jpa:
    database: MySQL
    database-platform: org.hibernate.dialect.MySQLDialect
    show-sql: true
    hibernate:
      ddl-auto: update
```

### 6. java.lang.UnsupportedOperationException: io.lettuce.core.output.ValueOutput does not support set(long)

```java
        // 注意，这里必须使用Long返回类型，因为redis中的Integer对应java中的Long
        RedisScript<Long> defaultRedisScript = new DefaultRedisScript<>(luaScript, Long.class);

```

### 7. Spring Boot [3.4.0] is not compatible with this Spring Cloud release train

SpringCloud应用启动失败：

``` bash
***************************
APPLICATION FAILED TO START
***************************

Description:

Your project setup is incompatible with our requirements due to following reasons:

- Spring Boot [3.4.0] is not compatible with this Spring Cloud release train


Action:

Consider applying the following actions:

- Change Spring Boot version to one of the following versions [3.2.x, 3.3.x] .
You can find the latest Spring Boot versions here [https://spring.io/projects/spring-boot#learn]. 
If you want to learn more about the Spring Cloud Release train compatibility, you can visit this page [https://spring.io/projects/spring-cloud#overview] and check the [Release Trains] section.
If you want to disable this check, just set the property [spring.cloud.compatibility-verifier.enabled=false]
```

根因就是SpringBoot和SpringCloud版本不匹配，可以点击错误中的连接查看配套版本。下面这两个版本是配套的。

``` xml
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

```

### 8. 微服务模块中通过RestTemplate调用其他服务时报错，java.net.UnknownHostException: xxxservice

原因是因为RestTemplate是直接在类中new的，不是专门定义的Bean。

通过Bean 定义的 RestTemplate，可以很好地与其他 Spring 组件集成。例如，在使用服务发现（如 Eureka）和负载均衡（如 Ribbon）的微服务架构中，通过@LoadBalanced注解标记的RestTemplate（定义为 Bean）可以自动与 Ribbon 集成，实现基于服务名称的负载均衡调用。 而通过new 的 RestTemplate 没有集成Ribbon，因而找不到微服务。

``` java
@Configuration
public class AppConfig {
    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```