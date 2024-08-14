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