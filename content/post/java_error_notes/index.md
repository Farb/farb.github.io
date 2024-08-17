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