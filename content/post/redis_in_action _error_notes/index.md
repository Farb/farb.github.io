---
title: 基于Docker的Redis实战--报错总结篇
description:
slug: docker-redis-error
date: 2024-07-28
image: 
categories:
    - Redis
tags:
    - redis
    - docker
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---

### Can't open or create append-only dir appendonlydir: Permission denied


redis7配置sentinel时报错 Can't resolve instance hostname.

*** FATAL CONFIG FILE ERROR (Redis 7.2.5) ***
sentinel monitor master localhost 6379 2
# 哨兵节点监控主服务器，主机ip是172.17.0.2，端口是6379，集群中至少2个哨兵节点才能确定该主服务器是否故障

Reading the configuration file, at line 2
>>> 'sentinel monitor redis-master localhost 6379 2'
Can't resolve instance hostname.

# 加上这句解决问题
sentinel resolve-hostnames yes
