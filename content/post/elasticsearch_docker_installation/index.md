---
title: 基于Docker安装ElasticSearch
description:
slug: elasticsearch_docker_installation
date: 2024-12-15
image: 
categories:
    - ElasticSearch
tags:
    - Docker
    - ElasticSearch
    - Kibana
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---

### 安装步骤

#### 创建网络

```bash
 # 因为需要部署kibana容器，因此需要让es和kibana容器互联。
 PS C:\Users\farbg> docker network create es-net
800d4c36198d27f6a96d08e27f7b7f08123e25eb2c3ec354ab9f03b629975248
```

#### 拉取镜像

```bash
# 注意：这里必须指定一个存在的版本号，否则可能会报一些奇怪的错。
docker pull elasticsearch:8.16.1
```

#### 启动容器

```bash

PS C:\Users\farbg> docker run -itd --restart=always --name es --network es-net -p 9200:9200 -p 9300:9300 --privileged -v D:\ArchitectPracticer\ElasticSearch/data:/usr/share/elasticsearch/data -v D:\ArchitectPracticer\ElasticSearch/plugins:/usr/share/elasticsearch/plugins -e "discovery.type=single-node" -e "ES_JAVA_OPTS=-Xms512m -Xmx512m" -e xpack.security.enabled=false elasticsearch:8.16.1

cebcc742cb1e857cec7ac96e0d58e7d7fe95a223b10713a811dcd96c3ff4fa06
```

**命令各参数含义分析**

1. docker run -itd：
-i 表示以交互模式运行容器，让容器的标准输入保持打开状态；-t 则是分配一个伪终端，通常二者配合使用可以方便进入容器内部进行交互操作。-d 是让容器在后台运行（--detach 的缩写），这样容器启动后不会占据当前终端，而是在后台默默运行，并且返回容器的 ID。
2. --restart=always：
设置容器的重启策略为始终重启。意味着无论容器因为何种原因退出（比如意外崩溃、手动停止等），Docker 都会自动重新启动该容器，有助于保证 Elasticsearch 服务的持续可用性，这对于生产环境或需要长时间稳定运行的场景很重要。
3. --name es：
给创建的容器指定一个名称为 “es”，后续对该容器进行操作（如查看状态、停止、启动、进入等）时，可以直接使用这个名称来指代容器，相较于使用容器 ID 更加方便易记。
4. --network es-net：
将容器连接到名为 “es-net” 的网络中。如果这个网络不存在，Docker 会自动创建它。自定义网络方便容器之间进行通信，并且可以更灵活地配置网络相关的规则和参数，例如在构建 Elasticsearch 集群等场景下，多个相关容器可通过此网络相互交互。
5. -p 9200:9200 -p 9300:9300：
这是端口映射的设置，-p （--publish 的缩写）。前面的数字代表主机（即运行 Docker 的物理机或虚拟机）上的端口，后面的数字代表容器内部的端口。所以 9200:9200 表示将容器内 Elasticsearch 用于 HTTP 通信的 9200 端口映射到主机的 9200 端口，方便外部客户端通过主机的 9200 端口访问容器内的 Elasticsearch 服务；9300:9300 则是把容器内用于节点间通信的 9300 端口映射到主机的 9300 端口，在集群模式下节点间通过这个端口进行交互，即便这里配置的是单节点模式，该端口映射也是必要的。
6. --privileged：
赋予容器特权模式，使得容器内的进程拥有和主机 root 用户几乎相同的权限。不过使用这个选项要谨慎，因为它会带来一定的安全风险，一般在需要容器访问主机某些底层设备或执行特殊系统调用等情况下才使用，比如 Elasticsearch 可能需要更高权限来进行一些诸如内存映射等底层操作时可能会用到。
7. -v D:\ArchitectPracticer\ElasticSearch/data:/usr/share/elasticsearch/data：
这是数据卷挂载的配置，-v （--volume 的缩写）。它将主机上的 D:\ArchitectPracticer\ElasticSearch/data 目录挂载到容器内的 /usr/share/elasticsearch/data 目录，目的是实现数据的持久化存储。这样即使容器被删除、重新创建等情况发生，Elasticsearch 存储的数据（比如索引数据等）依然保留在主机对应的目录中，不会丢失。
8. -v D:\ArchitectPracticer\ElasticSearch/plugins:/usr/share/elasticsearch/plugins：
同样是数据卷挂载操作，把主机的 D:\ArchitectPracticer\ElasticSearch/plugins 目录挂载到容器内的 /usr/share/elasticsearch/plugins 目录，方便对 Elasticsearch 插件进行管理和安装，插件相关的数据也能持久化保存在主机指定目录里。
9. -e "discovery.type=single-node"：
通过环境变量（-e，--env 的缩写）来设置 Elasticsearch 的发现类型为单节点模式。在这种模式下，Elasticsearch 不会去主动寻找其他节点来组成集群，适合在开发、测试环境或者简单的单机使用场景中应用，简化了配置和运行过程。
10. -e "ES_JAVA_OPTS=-Xms512m -Xmx512m"：
也是设置环境变量，用于指定 Elasticsearch 运行时 Java 虚拟机（JVM）的参数。这里将 JVM 的初始堆内存大小（-Xms）和最大堆内存大小（-Xmx）都设置为 512m，即 512MB，以此来控制 Elasticsearch 在运行过程中的内存使用情况，避免因内存占用过高或过低而出现问题。
11. -e xpack.security.enabled=false：
关闭Elasticsearch 的安全功能，默认为 true，表示启用安全功能，需要配置用户名密码等安全机制。
12. elasticsearch:8.16.1：
明确指定要运行的 Docker 镜像名称及版本号，这里表示运行 Elasticsearch 的 Docker 镜像，版本为 8.16.1。


#### 验证安装结果

浏览器访问http://localhost:9200，看到如下结果，说明安装成功。

``` json
{
    "name": "4d3e2d770e4a",
    "cluster_name": "docker-cluster",
    "cluster_uuid": "aUomG6AvRBanbkaizGncKA",
    "version": {
        "number": "8.16.1",
        "build_flavor": "default",
        "build_type": "docker",
        "build_hash": "ffe992aa682c1968b5df375b5095b3a21f122bf3",
        "build_date": "2024-11-19T16:00:31.793213192Z",
        "build_snapshot": false,
        "lucene_version": "9.12.0",
        "minimum_wire_compatibility_version": "7.17.0",
        "minimum_index_compatibility_version": "7.0.0"
    },
    "tagline": "You Know, for Search"
}
```

#### 基于Docker安装Kibana

**拉取镜像**

```bash
PS C:\Users\farbg> docker pull kibana:8.16.1
```

**运行容器**

```bash
docker run -itd --restart=always --name kibana --network es-net -p 5601:5601 -e ELASTICSEARCH_HOSTS=http://es:9200 kibana:8.16.1
```
**验证安装结果**
访问http://localhost:5601/，发现可以正常打开Kibana。

#### 基于Docker安装IK分词器

```bash
PS C:\Users\farbg> docker exec -it es /bin/bash

elasticsearch@4d3e2d770e4a:~$ bin/elasticsearch-plugin install https://get.infini.cloud/elasticsearch/analysis-ik/8.16.1

-> Installing https://get.infini.cloud/elasticsearch/analysis-ik/8.16.1
-> Downloading https://get.infini.cloud/elasticsearch/analysis-ik/8.16.1
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@     WARNING: plugin requires additional permissions     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
* java.net.SocketPermission * connect,resolve
See https://docs.oracle.com/javase/8/docs/technotes/guides/security/permissions.html
for descriptions of what these permissions allow and the associated risks.

Continue with installation? [y/N]y
-> Installed analysis-ik
-> Please restart Elasticsearch to activate any plugins installed
```

### 参考文章

基于Docker安装Elasticsearch【保姆级教程、内含图解】 https://blog.csdn.net/Acloasia/article/details/130683934

