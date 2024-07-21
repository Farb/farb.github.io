---
title: 基于Docker的Redis实战--构建Redis开发环境
description:
slug: redis_in_action_01_build_env
date: 2024-07-13
image: 
categories:
    - Redis
tags:
    - redis
    - docker
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---

下载Docker desktop，配置可用镜像地址。

docker 镜像可用地址： https://www.aabcc.top/archives/m7NPfx1D   

Redis客户端AnotherRedisDesktopManager： https://gitee.com/qishibo/AnotherRedisDesktopManager/releases

### 必要的docker技能
#### docker镜像相关的命令
``` sh
# 查看本机所有镜像
docker images 

# 拉取镜像,imageName为镜像名，比如ubuntu,tag为标签，如果没有指定，默认为latest
docker pull imageName:tag

# 删除本地镜像,可指定镜像名和标签，也可以指定镜像Id
docker rmi [imageName:tag | imageId]

```
#### docker容器相关的命令

``` sh
#基于镜像创建容器并运行容器
# -it 表示终端交互式操作 （interactive terminal）
#  ubuntu:latest指定待运行的镜像
# /bin/bash 表示容器启动后要执行的命令
docker run -it ubuntu:latest /bin/bash

# 查看运行中的容器
docker ps

# 查看所有的容器
docker ps -a

# 停止容器
docker stop containterId

# 启动容器
docker start containerName

# 删除容器,删除容器前必须停止容器，否则报错
docker rm containerId

# 容器重命名(docker run之后，如果不指定--name参数，则会随机生成一个容器名)
docker rename oldName newName

# 进入容器后，通过第一次exit可以退出容器
```

### 安装和配置基于docker的Redis环境

``` sh
# 下载最新redis镜像
docker pull redis:latest

# 查看镜像
docker images

# 启动redis容器
# -it表示终端交互式操作，d表示后台运行
# --name 指定容器名称
# -p 指定容器的6379端口映射到宿主机（运行docker的机器）6379端口
# redis:latest 为启动容器的镜像
docker run -itd --name firstRedis -p 6379:6379 redis:latest


# 查看容器日志 
docker logs firstRedis

# 进入容器
# docker exec 表示在运行的容器中执行命令
# -it 表示以交互式终端执行命令
# firstRedis为容器名
# /bin/bash 为需要执行的命令
docker exec -it firstRedis /bin/bash

# 通过redis-cli命令进入redis客户端，可以执行set ,get命令

# 通过两次exit可以退出到windows命令行。第一次exit是退出redis-cli客户端到容器内部，第二次exit是退出容器。

# 停止容器，可以通过docker ps -a 对比容器前后状态
docker stop firstRedis

# 启动容器
docker start firstRedis

# 重启容器
docker restart firstRedis

## start和restart的区别是：start会挂载容器所关联的文件系统，restart不会。如果更改了redis启动时需要加载的配置项参数，那么重启时需要先stop，再start。

# 查看redis版本
docker start firstRedis
docker exec -it firstRedis /bin/bash

# 查看redis客户端版本
redis-cli --version

# 查看reis服务端版本
redis-server --version

# 如果要停止redis服务，可以通过exit退出容器，也可以通过redis-cli的shutdown命令关闭服务。
```
