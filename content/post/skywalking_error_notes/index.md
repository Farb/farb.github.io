---
title: SkyWalking 报错记录集
description:
slug: skywalking_error_notes
date: 2024-12-19
image: 
categories:
    - SkyWalking
tags:
    - SkyWalking
    - error_notes
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---

### 1. SkyWalking启动报错 Caused by: io.netty.channel.AbstractChannel$AnnotatedConnectException: finishConnect(..) failed: Connection refused: elasticsearch/localhost:9200

skywalking源码地址：https://github.com/apache/skywalking

经调查，发现是skywalking和elasticsearch的版本不匹配，需要修改skywalking的版本，或者修改elasticsearch的版本。

elasticsearch只是skywalking的一种存储方式，所以，skywalking需要适配es。所以不是每个版本的skywalking和es都能正常工作。

报错的docker_compose.yml为：

```yaml
services:
  elasticsearch:
    # image: docker.elastic.co/elasticsearch/elasticsearch-oss:${ES_VERSION}
    image: elasticsearch:8.16.1
    container_name: elasticsearch
    # --restart=always ： 开机启动，失败也会一直重启；
    # --restart=on-failure:10 ： 表示最多重启10次
    restart: always
    ports:
      - "9200:9200"
      - "9300:9300"
    healthcheck:
      test: [ "CMD-SHELL", "curl --silent --fail localhost:9200/_cluster/health || exit 1" ]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 10s
    environment:
      - discovery.type=single-node
      # 锁定物理内存地址，防止elasticsearch内存被交换出去,也就是避免es使用swap交换分区，频繁的交换，会导致IOPS变高；
      - bootstrap.memory_lock=true
      # 设置时区
      - TZ=Asia/Shanghai
      # - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1

  oap:
    image: apache/skywalking-oap-server:10.1.0
    container_name: oap
    restart: always
    # 设置依赖的容器
    depends_on:
      elasticsearch:
        condition: service_healthy
    # 关联ES的容器，通过容器名字来找到相应容器，解决IP变动问题
    links:
      - elasticsearch
    # 端口映射
    ports:
      - "11800:11800"
      - "12800:12800"
    # 监控检查
    healthcheck:
      test: [ "CMD-SHELL", "/skywalking/bin/swctl ch" ]
      # 每间隔30秒执行一次
      interval: 30s
      # 健康检查命令运行超时时间，如果超过这个时间，本次健康检查就被视为失败；
      timeout: 10s
      # 当连续失败指定次数后，则将容器状态视为 unhealthy，默认 3 次。
      retries: 3
      # 应用的启动的初始化时间，在启动过程中的健康检查失效不会计入，默认 0 秒。
      start_period: 10s
    environment:
      # 指定存储方式
      SW_STORAGE: elasticsearch
      # 指定存储的地址
      SW_STORAGE_ES_CLUSTER_NODES: elasticsearch:9200
      SW_HEALTH_CHECKER: default
      SW_TELEMETRY: prometheus
      TZ: Asia/Shanghai
      # JAVA_OPTS: "-Xms2048m -Xmx2048m"

  ui:
    image: apache/skywalking-ui:10.1.0
    container_name: skywalking-ui
    restart: always
    depends_on:
      oap:
        condition: service_healthy
    links:
      - oap
    ports:
      - "8088:8080"
    environment:
      SW_OAP_ADDRESS: http://oap:12800
      TZ: Asia/Shanghai

```

可以正常工作的docker_compose.yml为：

```yml
services:
  elasticsearch:
    # image: docker.elastic.co/elasticsearch/elasticsearch-oss:${ES_VERSION}
    image: elasticsearch:7.9.0
    container_name: elasticsearch
    # --restart=always ： 开机启动，失败也会一直重启；
    # --restart=on-failure:10 ： 表示最多重启10次
    restart: always
    ports:
      - "9200:9200"
      - "9300:9300"
    healthcheck:
      test: [ "CMD-SHELL", "curl --silent --fail localhost:9200/_cluster/health || exit 1" ]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 10s
    environment:
      - discovery.type=single-node
      # 锁定物理内存地址，防止elasticsearch内存被交换出去,也就是避免es使用swap交换分区，频繁的交换，会导致IOPS变高；
      - bootstrap.memory_lock=true
      # 设置时区
      - TZ=Asia/Shanghai
      # - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1

  oap:
    image: apache/skywalking-oap-server:8.9.1
    container_name: oap
    restart: always
    # 设置依赖的容器
    depends_on:
      elasticsearch:
        condition: service_healthy
    # 关联ES的容器，通过容器名字来找到相应容器，解决IP变动问题
    links:
      - elasticsearch
    # 端口映射
    ports:
      - "11800:11800"
      - "12800:12800"
    # 监控检查
    healthcheck:
      test: [ "CMD-SHELL", "/skywalking/bin/swctl ch" ]
      # 每间隔30秒执行一次
      interval: 30s
      # 健康检查命令运行超时时间，如果超过这个时间，本次健康检查就被视为失败；
      timeout: 10s
      # 当连续失败指定次数后，则将容器状态视为 unhealthy，默认 3 次。
      retries: 3
      # 应用的启动的初始化时间，在启动过程中的健康检查失效不会计入，默认 0 秒。
      start_period: 10s
    environment:
      # 指定存储方式
      SW_STORAGE: elasticsearch
      # 指定存储的地址
      SW_STORAGE_ES_CLUSTER_NODES: elasticsearch:9200
      SW_HEALTH_CHECKER: default
      SW_TELEMETRY: prometheus
      TZ: Asia/Shanghai
      # JAVA_OPTS: "-Xms2048m -Xmx2048m"

  ui:
    image: apache/skywalking-ui:8.9.1
    container_name: skywalking-ui
    restart: always
    depends_on:
      oap:
        condition: service_healthy
    links:
      - oap
    ports:
      - "8088:8080"
    environment:
      SW_OAP_ADDRESS: http://oap:12800
      TZ: Asia/Shanghai

```

执行成功后，可以看到如下输出：

```bash
PS C:\Users\farbg\Desktop> docker-compose -f .\skywalking_docker-compose2.yml up -d
[+] Running 4/4
 ✔ Network desktop_default Created    0.0s 
 ✔ Container elasticsearch  Healthy   30.9s 
 ✔ Container oap            Healthy   61.7s 
 ✔ Container skywalking-ui  Started 
```