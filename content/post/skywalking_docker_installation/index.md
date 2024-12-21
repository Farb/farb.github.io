---
title: 基于Docker安装SkyWalking
description:
slug: skywalking_docker_installation
date: 2024-12-20
image: 
categories:
    - SkyWalking
tags:
    - Docker
    - SkyWalking
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---

### 安装步骤

1. 先安装elasticsearch，作为skywalking的存储
2. 再安装oap-server，作为skywalking的服务端
3. 最后安装ui，作为skywalking的前端

涉及三个镜像服务，使用dockercompose来部署比较方便。

#### 创建dockercompose文件

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

#### 执行dockercompose

执行成功后，可以看到如下输出：

```bash
# 如果文件名是标准的docker-compose.yml，直接执行docker-compose up -d即可
PS C:\Users\farbg\Desktop> docker-compose -f .\skywalking_docker-compose2.yml up -d
[+] Running 4/4
 ✔ Network desktop_default Created    0.0s 
 ✔ Container elasticsearch  Healthy   30.9s 
 ✔ Container oap            Healthy   61.7s 
 ✔ Container skywalking-ui  Started 
```

#### 验证容器是否启动成功

访问 http://localhost:8088/ ，可以看到skywalking管理界面。

![](https://s3.bmp.ovh/imgs/2024/12/21/61c57472446e874c.png)

### 安装及配置探针 agent
下载地址： https://skywalking.apache.org/downloads/#Agents

idea中在对应的服务配置jvm参数如下：

```
-javaagent:D:\ArchitectPracticer\MicroServiceTools\apache-skywalking-java-agent-9.3.0\skywalking-agent\skywalking-agent.jar
-Dskywalking.agent.service_name=FirstServiceDemo
-Dskywalking.collector.backend_service=127.0.0.1:11800
```

-javaagent:D:\ArchitectPracticer\MicroServiceTools\apache-skywalking-java-agent-9.3.0\skywalking-agent\skywalking-agent.jar：指定Skywalking代理（Java探针）的路径，它负责收集应用程序的性能指标和调用链路数据。
-Dskywalking.agent.service_name=FirstServiceDemo：设置当前Java应用在Skywalking中的服务名称为“FirstServiceDemo”，便于在监控界面中识别和区分不同服务。
-Dskywalking.collector.backend_service=127.0.0.1:11800：配置Skywalking后端Collector服务地址和端口，该Java应用通过此地址将收集到的数据上报至Skywalking OAP Server进行分析和存储。这里设置的是本地回环地址（localhost），端口号为11800。

### 测试接口

简单写了一个接口，查询一个用户信息

```java
@RestController
public class TestController {
    private final IUserAccountService userAccountService;

    public TestController(IUserAccountService userAccountService) {
        this.userAccountService = userAccountService;
    }

    @GetMapping("/test")
    public String test() {
        return test2();
    }

    @GetMapping("/test2")
    public String test2() {
        return this.userAccountService.getById(1).toString();
    }
}

```
可以在 **追踪**菜单中查看到调用链路
![](https://s3.bmp.ovh/imgs/2024/12/21/da422751c53136a4.png)

点击某个链路，还能看到具体的详情，比如可以看到查询的sql
![](https://s3.bmp.ovh/imgs/2024/12/21/6d32e9b58e12b4bf.png)

还可以查看拓扑图
![](https://s3.bmp.ovh/imgs/2024/12/21/4dd895eb5f0735fd.png)

### 参考文章

- https://blog.51cto.com/u_16213348/12680103
- https://blog.csdn.net/qq_31762741/article/details/136778188
- https://blog.csdn.net/Li_WenZhang/article/details/140992333