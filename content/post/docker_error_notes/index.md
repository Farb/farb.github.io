---
title: Docker报错记录集
description:
slug: docker_error_notes
date: 2024-09-08
image: 
categories:
    - Docker
tags:
    - Docker
    - error_notes
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---

### 1. docker: Error response from daemon: Ports are not available

```bash
D:\DevTools\java\Mycat-Server-1.6.76-release-2020-11-2\target>docker run --name mycat -p 8066:8066 -p 9066:9066 -v /usr/local/mycat/conf/:/usr/local/mycat/conf/ -v /usr/local/mycat/logs/:/usr/local/mycat/logs/ -d mycat:1.6.7.6
0d1c9dad51d7147e31ec0fdf42480bd5bf2c0d798507ee20f41b130184ff9a66
docker: Error response from daemon: Ports are not available: exposing port TCP 0.0.0.0:8066 -> 0.0.0.0:0: listen tcp 0.0.0.0:8066: bind: An attempt was made to access a socket in a way forbidden by its access permissions.

```

解决方法：使用netstat -aon | findstr 查看并没有占用

``` bash
net stop winnat
net start winnat

# 然后重新创建并运行容器即可
```
