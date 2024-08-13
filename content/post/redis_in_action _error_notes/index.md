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

### 1:M 11 Aug 2024 15:34:39.576 # Unable to obtain the AOF file appendonly-clusterMaster4.aof.3.incr.aof length. stat: No such file or directory
不需要配置了appendfilename appendonly-clusterMaster4.aof


### Failed trying to load the MASTER synchronization DB from disk, check server logs.

根据报错提示，回到主节点可以看到错误日志,可以看到和从节点的连接丢失，没有同步成功。
```log
1:M 12 Aug 2024 15:56:25.216 * Starting BGSAVE for SYNC with target: replicas sockets
1:M 12 Aug 2024 15:56:25.222 * Background RDB transfer started by pid 220
220:C 12 Aug 2024 15:56:25.222 * Fork CoW for RDB: current 0 MB, peak 0 MB, average 0 MB
1:M 12 Aug 2024 15:56:25.225 * Diskless rdb transfer, done reading from pipe, 1 replicas still up.
1:M 12 Aug 2024 15:56:25.231 * Connection with replica 172.17.0.9:16382 lost.
1:M 12 Aug 2024 15:56:25.235 * Replica 172.17.0.9:16382 asks for synchronization
1:M 12 Aug 2024 15:56:25.235 * Full resync requested by replica 172.17.0.9:16382
1:M 12 Aug 2024 15:56:25.236 * Current BGSAVE has socket target. Waiting for next BGSAVE for SYNC
1:M 12 Aug 2024 15:56:25.324 * Background RDB transfer terminated with success
```

修改redis.config中的配置,将主从服务器的属性repl-diskless-load设置为on-empty-db即可

```sh
# Replica can load the RDB it reads from the replication link directly from the
# socket, or store the RDB to a file and read that file after it was completely
# received from the master.
#
# In many cases the disk is slower than the network, and storing and loading
# the RDB file may increase replication time (and even increase the master's
# Copy on Write memory and replica buffers).
# However, parsing the RDB file directly from the socket may mean that we have
# to flush the contents of the current database before the full rdb was
# received. For this reason we have the following options:
#
# "disabled"    - Don't use diskless load (store the rdb file to the disk first)
# "on-empty-db" - Use diskless load only when it is completely safe.
# "swapdb"      - Keep current db contents in RAM while parsing the data directly
#                 from the socket. Replicas in this mode can keep serving current
#                 data set while replication is in progress, except for cases where
#                 they can't recognize master as having a data set from same
#                 replication history.
#                 Note that this requires sufficient memory, if you don't have it,
#                 you risk an OOM kill.
repl-diskless-load disabled

```

然后重启主从服务器，再次查看主节点日志，发现同步成功
```log
1:C 12 Aug 2024 15:56:32.400 # WARNING: Changing databases number from 16 to 1 since we are in cluster mode
1:C 12 Aug 2024 15:56:32.401 * oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
1:C 12 Aug 2024 15:56:32.402 * Redis version=7.2.5, bits=64, commit=00000000, modified=0, pid=1, just started
1:C 12 Aug 2024 15:56:32.403 * Configuration loaded
1:M 12 Aug 2024 15:56:32.404 * monotonic clock: POSIX clock_gettime
1:M 12 Aug 2024 15:56:32.405 * Running mode=cluster, port=6382.
1:M 12 Aug 2024 15:56:32.407 * Node configuration loaded, I'm 8ae631285a8532aaeb324b404ab48352a25658ff
1:M 12 Aug 2024 15:56:32.408 * Server initialized
1:M 12 Aug 2024 15:56:32.414 * Reading RDB base file on AOF loading...
1:M 12 Aug 2024 15:56:32.416 * Loading RDB produced by version 7.2.5
1:M 12 Aug 2024 15:56:32.416 * RDB age 961 seconds
1:M 12 Aug 2024 15:56:32.417 * RDB memory usage when created 1.59 Mb
1:M 12 Aug 2024 15:56:32.418 * RDB is base AOF
1:M 12 Aug 2024 15:56:32.419 * Done loading RDB, keys loaded: 0, keys expired: 0.
1:M 12 Aug 2024 15:56:32.420 * DB loaded from base file appendonly-clusterMaster4.2.base.rdb: 0.006 seconds
1:M 12 Aug 2024 15:56:32.422 * DB loaded from incr file appendonly-clusterMaster4.2.incr.aof: 0.001 seconds
1:M 12 Aug 2024 15:56:32.422 * DB loaded from append only file: 0.012 seconds
1:M 12 Aug 2024 15:56:32.424 * Opening AOF incr file appendonly-clusterMaster4.2.incr.aof on server start
1:M 12 Aug 2024 15:56:32.425 * Ready to accept connections tcp
1:M 12 Aug 2024 15:56:33.495 * Replica 172.17.0.9:16382 asks for synchronization
1:M 12 Aug 2024 15:56:33.496 * Partial resynchronization not accepted: Replication ID mismatch (Replica asked for 'a4e7ecbdc1e68bb729f8e293cdd37af350bcc946', my replication IDs are 'a3393f87f433bfd8bc8c205a0911ffa83a72831d' and '0000000000000000000000000000000000000000')
1:M 12 Aug 2024 15:56:33.497 * Replication backlog created, my new replication IDs are '16dd21acae0e5ba18ee4699070c0046ef26ac227' and '0000000000000000000000000000000000000000'
1:M 12 Aug 2024 15:56:33.498 * Delay next BGSAVE for diskless SYNC
1:M 12 Aug 2024 15:56:34.442 * Cluster state changed: ok
1:M 12 Aug 2024 15:56:38.373 * Starting BGSAVE for SYNC with target: replicas sockets
1:M 12 Aug 2024 15:56:38.375 * Background RDB transfer started by pid 20
20:C 12 Aug 2024 15:56:38.375 * Fork CoW for RDB: current 0 MB, peak 0 MB, average 0 MB
1:M 12 Aug 2024 15:56:38.376 * Diskless rdb transfer, done reading from pipe, 1 replicas still up.
1:M 12 Aug 2024 15:56:38.388 * Background RDB transfer terminated with success
1:M 12 Aug 2024 15:56:38.389 * Streamed RDB transfer with replica 172.17.0.9:16382 succeeded (socket). Waiting for REPLCONF ACK from replica to enable streaming
1:M 12 Aug 2024 15:56:38.390 * Synchronization with replica 172.17.0.9:16382 succeeded

```