---
title: 基于Docker的Redis实战--实践Redis服务器和客户端的操作
description:
slug: redis_in_action_04_server_and_client
date: 2024-07-22
image: 
categories:
    - Redis
tags:
    - redis
    - docker
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---


### Redis服务器管理客户端的命令
#### 获取或设置客户端的名字
```sh
PS D:\code\blogs\farb.github.io> docker start firstRedis
firstRedis
PS D:\code\blogs\farb.github.io> docker exec -it firstRedis /bin/bash
root@380d98e7beb9:/data# redis-cli
127.0.0.1:6379> client getname # 获取客户端名称
(nil)
127.0.0.1:6379> client setname client1 # 设置客户端名称
OK
127.0.0.1:6379> client getname
"client1"
```

#### 通过client list查看客户端信息
```sh
127.0.0.1:6379> client list
id=3 addr=127.0.0.1:32982 laddr=127.0.0.1:6379 fd=8 name=client1 age=133 idle=0 flags=N db=0 sub=0 psub=0 ssub=0 multi=-1 qbuf=26 qbuf-free=20448 argv-mem=10 multi-mem=0 rbs=1024 rbp=0 obl=0 oll=0 omem=0 tot-mem=22426 events=r cmd=client|list user=default redir=-1 resp=2 lib-name= lib-ver=

# id:客户端的编号
# addr:客户端的地址
# name:客户端名称
# age:客户端的连接时长，单位是s
# idle：客户端的空闲时长，单位是s
# db：客户端用到的服务器数据库索引号，默认每个redis服务器有16个数据库，且默认使用0号数据库
# cmd: 客户端最近执行的命令
# user:登录服务器用到的用户名

# 如果有多个客户端连接，则返回多行
```

#### 通过client pause暂停客户端的命令
```sh
 client pause timeout [WRITE|ALL]
# 如果服务端压力过大，则可暂停执行客户端的命令
# timeout表示暂停的毫秒数

127.0.0.1:6379> client pause 10000
OK
127.0.0.1:6379> set name val2
OK
(3.89s)
# 客户端暂停10s后，执行set命令时不会立即执行，而是会卡住，最后返回结果和卡住的时间。

127.0.0.1:6379> client pause 10000
OK
127.0.0.1:6379> get name
"val2"
(2.71s)
# 默认参数为ALL，读写都会暂停
# 如果只指定写的命令暂停，那么读的命令不会暂停
127.0.0.1:6379> client pause 10000 write
OK
127.0.0.1:6379> get name
"val2"
127.0.0.1:6379> set name val3
OK
(4.15s)
```

#### 通过client kill中断客户端的连接
```sh
client kill ip:port|[ID client-id]|[TYPE NORMAL|MASTER|SLAVE|REPLICA|PUBSUB]|[USER username]|[ADDR ip:port]|[LADDR ip:port]|[SKIPME YES|NO] [[ID client-id]|[TYPE NORMAL|MAS
# 可以看到终端客户端的连接支持很多参数

127.0.0.1:6379> client list
id=3 addr=127.0.0.1:32982 laddr=127.0.0.1:6379 fd=8 name=client1 age=1420 idle=391 flags=N db=0 sub=0 psub=0 ssub=0 multi=-1 qbuf=0 qbuf-free=0 argv-mem=0 multi-mem=0 rbs=1024 rbp=0 obl=0 oll=0 omem=0 tot-mem=1928 events=r cmd=set user=default redir=-1 resp=2 lib-name= lib-ver=
id=4 addr=127.0.0.1:41212 laddr=127.0.0.1:6379 fd=9 name= age=179 idle=179 flags=N db=0 sub=0 psub=0 ssub=0 multi=-1 qbuf=0 qbuf-free=0 argv-mem=0 multi-mem=0 rbs=1024 rbp=0 obl=0 oll=0 omem=0 tot-mem=1928 events=r cmd=command|docs user=default redir=-1 resp=2 lib-name= lib-ver=
id=5 addr=127.0.0.1:41528 laddr=127.0.0.1:6379 fd=10 name= age=8 idle=0 flags=N db=0 sub=0 psub=0 ssub=0 multi=-1 qbuf=26 qbuf-free=20448 argv-mem=10 multi-mem=0 rbs=1024 rbp=0 obl=0 oll=0 omem=0 tot-mem=22426 events=r cmd=client|list user=default redir=-1 resp=2 lib-name= lib-ver=
127.0.0.1:6379> client kill 127.0.0.1:32982
OK
127.0.0.1:6379> client kill 127.0.0.1:41212
OK
127.0.0.1:6379> client list
id=5 addr=127.0.0.1:41528 laddr=127.0.0.1:6379 fd=10 name= age=190 idle=0 flags=N db=0 sub=0 psub=0 ssub=0 multi=-1 qbuf=26 qbuf-free=20448 argv-mem=10 multi-mem=0 rbs=1024 rbp=0 obl=0 oll=0 omem=0 tot-mem=22426 events=r cmd=client|list user=default redir=-1 resp=2 lib-name= lib-ver=

# 查看中断的客户端，执行get命令时，发现报错。
127.0.0.1:6379> get name
Error: Broken pipe
not connected>
```

#### shutdown关闭服务器和客户端
```sh
127.0.0.1:6379> shutdown
# shutdown 会终止服务器上的所有客户端连接，并终止服务器
```

### 查看Redis服务器的详细信息
#### info命令查看服务器信息
```sh
127.0.0.1:6379> info
# Server 服务端信息
redis_version:7.2.5
redis_git_sha1:00000000
redis_git_dirty:0
redis_build_id:c2b7a5cd72a5634f
redis_mode:standalone
os:Linux 5.15.153.1-microsoft-standard-WSL2 x86_64
arch_bits:64
monotonic_clock:POSIX clock_gettime
multiplexing_api:epoll
atomicvar_api:c11-builtin
gcc_version:12.2.0
process_id:1
process_supervised:no
run_id:7284acf78bb2982dc66fec7c06a0fcb79ab0865a
tcp_port:6379
server_time_usec:1721658345403020
uptime_in_seconds:21
uptime_in_days:0
hz:10
configured_hz:10
lru_clock:10382313
executable:/data/redis-server
config_file:
io_threads_active:0
listener0:name=tcp,bind=*,bind=-::*,port=6379

# Clients 已连接的客户端信息
connected_clients:1
cluster_connections:0
maxclients:10000
client_recent_max_input_buffer:8
client_recent_max_output_buffer:0
blocked_clients:0
tracking_clients:0
clients_in_timeout_table:0
total_blocking_keys:0
total_blocking_keys_on_nokey:0

# Memory 服务器内存的相关信息
used_memory:917080
used_memory_human:895.59K
used_memory_rss:14802944
used_memory_rss_human:14.12M
used_memory_peak:1135136
used_memory_peak_human:1.08M
used_memory_peak_perc:80.79%
used_memory_overhead:868656
used_memory_startup:865744
used_memory_dataset:48424
used_memory_dataset_perc:94.33%
allocator_allocated:1456640
allocator_active:1572864
allocator_resident:7090176
total_system_memory:16626606080
total_system_memory_human:15.48G
used_memory_lua:31744
used_memory_vm_eval:31744
used_memory_lua_human:31.00K
used_memory_scripts_eval:0
number_of_cached_scripts:0
number_of_functions:0
number_of_libraries:0
used_memory_vm_functions:32768
used_memory_vm_total:64512
used_memory_vm_total_human:63.00K
used_memory_functions:184
used_memory_scripts:184
used_memory_scripts_human:184B
maxmemory:0
maxmemory_human:0B
maxmemory_policy:noeviction
allocator_frag_ratio:1.08
allocator_frag_bytes:116224
allocator_rss_ratio:4.51
allocator_rss_bytes:5517312
rss_overhead_ratio:2.09
rss_overhead_bytes:7712768
mem_fragmentation_ratio:16.55
mem_fragmentation_bytes:13908752
mem_not_counted_for_evict:0
mem_replication_backlog:0
mem_total_replication_buffers:0
mem_clients_slaves:0
mem_clients_normal:1928
mem_cluster_links:0
mem_aof_buffer:0
mem_allocator:jemalloc-5.3.0
active_defrag_running:0
lazyfree_pending_objects:0
lazyfreed_objects:0

# Persistence 持久化相关信息
loading:0
async_loading:0
current_cow_peak:0
current_cow_size:0
current_cow_size_age:0
current_fork_perc:0.00
current_save_keys_processed:0
current_save_keys_total:0
rdb_changes_since_last_save:0
rdb_bgsave_in_progress:0
rdb_last_save_time:1721658324
rdb_last_bgsave_status:ok
rdb_last_bgsave_time_sec:-1
rdb_current_bgsave_time_sec:-1
rdb_saves:0
rdb_last_cow_size:0
rdb_last_load_keys_expired:0
rdb_last_load_keys_loaded:16
aof_enabled:0
aof_rewrite_in_progress:0
aof_rewrite_scheduled:0
aof_last_rewrite_time_sec:-1
aof_current_rewrite_time_sec:-1
aof_last_bgrewrite_status:ok
aof_rewrites:0
aof_rewrites_consecutive_failures:0
aof_last_write_status:ok
aof_last_cow_size:0
module_fork_in_progress:0
module_fork_last_cow_size:0

# Stats 服务器相关的统计信息
total_connections_received:1
total_commands_processed:1
instantaneous_ops_per_sec:0
total_net_input_bytes:41
total_net_output_bytes:205205
total_net_repl_input_bytes:0
total_net_repl_output_bytes:0
instantaneous_input_kbps:0.00
instantaneous_output_kbps:0.00
instantaneous_input_repl_kbps:0.00
instantaneous_output_repl_kbps:0.00
 # 拒绝连接数，超过最大连接数后，连接会被拒绝
rejected_connections:0
sync_full:0
sync_partial_ok:0
sync_partial_err:0
expired_keys:0
expired_stale_perc:0.00
expired_time_cap_reached_count:0
expire_cycle_cpu_milliseconds:0
evicted_keys:0
evicted_clients:0
total_eviction_exceeded_time:0
current_eviction_exceeded_time:0
keyspace_hits:0
keyspace_misses:0
pubsub_channels:0
pubsub_patterns:0
pubsubshard_channels:0
latest_fork_usec:0
total_forks:0
migrate_cached_sockets:0
slave_expires_tracked_keys:0
active_defrag_hits:0
active_defrag_misses:0
active_defrag_key_hits:0
active_defrag_key_misses:0
total_active_defrag_time:0
current_active_defrag_time:0
tracking_total_keys:0
tracking_total_items:0
tracking_total_prefixes:0
unexpected_error_replies:0
total_error_replies:0
dump_payload_sanitizations:0
total_reads_processed:2
total_writes_processed:3
io_threaded_reads_processed:0
io_threaded_writes_processed:0
reply_buffer_shrinks:1
reply_buffer_expands:0
eventloop_cycles:213
eventloop_duration_sum:13918
eventloop_duration_cmd_sum:389
instantaneous_eventloop_cycles_per_sec:9
instantaneous_eventloop_duration_usec:62
acl_access_denied_auth:0
acl_access_denied_cmd:0
acl_access_denied_key:0
acl_access_denied_channel:0

# Replication 数据库主从复制相关的信息
role:master
connected_slaves:0
master_failover_state:no-failover
master_replid:35faf12e5068ef7be8ccb60da6198a1964e11907
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:0
second_repl_offset:-1
repl_backlog_active:0
repl_backlog_size:1048576
repl_backlog_first_byte_offset:0
repl_backlog_histlen:0

# CPU Redis服务器所在机器的相关信息
used_cpu_sys:0.009357
used_cpu_user:0.019368
used_cpu_sys_children:0.000207
used_cpu_user_children:0.000866
used_cpu_sys_main_thread:0.027521
used_cpu_user_main_thread:0.000000

# Modules

# Errorstats

# Cluster 集群相关信息
cluster_enabled:0

# Keyspace 包含了和数据库相关的统计信息，如键数量和超时时间等
db0:keys=16,expires=0,avg_ttl=0
```

#### 查看客户端连接状况
```sh
127.0.0.1:6379> info clients
# Clients
connected_clients:1 #已连接的客户端数
cluster_connections:0
maxclients:10000
client_recent_max_input_buffer:8
client_recent_max_output_buffer:0
blocked_clients:0
tracking_clients:0
clients_in_timeout_table:0
total_blocking_keys:0
total_blocking_keys_on_nokey:0
```

#### 观察最大连接数
```sh
# 通过info Clients可以看到默认的最大客户端连接数是10000
# 也可以通过config get获取
127.0.0.1:6379> config get maxclients
1) "maxclients"
2) "10000"

# 如果发现1w个连接不够用，可以设置大一些
 config set maxclients 200000 value 
```

#### 查看每秒执行多少指令
```sh
# Info stats中下列项表示每秒执行的操作
instantaneous_ops_per_sec:0

# 当发现集群里面Redis服务器的该数值过大或过小，就需要观察负载均衡的相关配置
# 当发现数据库压力过大而该值较小时，就需要调整缓存策略
```

#### 观察内存用量
Redis在内存中缓存数据，如果缓存数据过多，或者大量键没有设置过期时间，就会造成内存使用过大，从而导致OOM。
```sh
127.0.0.1:6379> info memory
# Memory
used_memory:964832
used_memory_human:942.22K
used_memory_rss:15634432
used_memory_rss_human:14.91M
used_memory_peak:1185104
used_memory_peak_human:1.13M
used_memory_peak_perc:81.41%
used_memory_overhead:868656
used_memory_startup:865744
used_memory_dataset:96176
used_memory_dataset_perc:97.06%
allocator_allocated:1795584
allocator_active:1949696
allocator_resident:7467008
total_system_memory:16626606080
total_system_memory_human:15.48G
used_memory_lua:31744
used_memory_vm_eval:31744
used_memory_lua_human:31.00K
used_memory_scripts_eval:0
number_of_cached_scripts:0
number_of_functions:0
number_of_libraries:0
used_memory_vm_functions:32768
used_memory_vm_total:64512
used_memory_vm_total_human:63.00K
used_memory_functions:184
used_memory_scripts:184
used_memory_scripts_human:184B
maxmemory:0
maxmemory_human:0B
maxmemory_policy:noeviction
allocator_frag_ratio:1.09
allocator_frag_bytes:154112
allocator_rss_ratio:3.83
allocator_rss_bytes:5517312
rss_overhead_ratio:2.09
rss_overhead_bytes:8167424
mem_fragmentation_ratio:16.56
mem_fragmentation_bytes:14690272
mem_not_counted_for_evict:0
mem_replication_backlog:0
mem_total_replication_buffers:0
mem_clients_slaves:0
mem_clients_normal:1928
mem_cluster_links:0
mem_aof_buffer:0
mem_allocator:jemalloc-5.3.0
active_defrag_running:0
lazyfree_pending_objects:0
lazyfreed_objects:0

# used_memory_human:操作系统分配给Redis多少内存
# used_memory_peak_human:1.13M Redis服务器用到的内存峰值
# used_memory_lua_human:31.00K Lua脚本所占用的内存
# used_memory_scripts_human:184B 脚本所占用的内存
# mem_clients_slaves:0 客户端因主从复制而使用的内存用量
```

#### 用command命令查看Redis命令
```sh
command 
# 返回服务器保护的所有命令信息
command count
# 统计当前Redis服务器命令个数
command info [command-name [command-name ...]] 
# 获取指定命令的详细信息

 command getkeys command [arg [arg ...]]
 # 获取指定命令的所有键
 127.0.0.1:6379> command getkeys set name farb
1) "name"
```

### 查看并修改服务器的常用配置
#### 查看服务器的配置
```sh
config get *
# 获取所有的服务器配置

# 获取以p开头的配置，支持通配符模糊匹配
127.0.0.1:6379> config get p*
 1) "proc-title-template"
 2) "{title} {listen-addr} {server-mode}"
 3) "proto-max-bulk-len"
 4) "536870912"
 5) "port"
 6) "6379"
 7) "protected-mode"
 8) "no"
 9) "pidfile"
10) ""
11) "propagation-error-behavior"
12) "ignore"
```

#### 修改服务器配置
``` sh
config set parameter value [parameter value ...]
# parameter是配置项，value是值，支持多个项一起配置
# 通过config set命令修改的配置无需重启即可生效

127.0.0.1:6379> config set requirepass 123456
OK
# 默认连接到Redis服务器不需要密码，此处设置密码

127.0.0.1:6379> quit
root@380d98e7beb9:/data# redis-cli
# 退出并重新登录客户端后，获取配置发现提示没有权限
127.0.0.1:6379> config get p*
(error) NOAUTH Authentication required.
# 使用auth password登录
127.0.0.1:6379> auth 123456
OK
127.0.0.1:6379> config get req*
1) "requirepass"
2) "123456"

# 先关闭Redis服务器，从而docker容器自动关闭
# 再启动容器并进入，然后查看密码，发现密码已清空
# 从而说明，config set虽然不需要重启服务器生效，但是重启服务器之后设置就会失效
127.0.0.1:6379> shutdown
PS D:\code\blogs\farb.github.io> docker start firstRedis
firstRedis
PS D:\code\blogs\farb.github.io> docker exec -it firstRedis /bin/bash
root@380d98e7beb9:/data# redis-cli          
127.0.0.1:6379> config get requirepass
1) "requirepass"
2) ""
```
#### 用config rewrite命令改写Redis配置文件
```sh
# 执行完config set之后，再执行config rewrite就会将配置保存到redis.conf文件
# 如果配置文件不存在就会报错
127.0.0.1:6379> config set requirepass 123456
OK
127.0.0.1:6379> config rewrite
(error) ERR The server is running without a config file
# 下面说明如何生成reids.conf
```
#### 启动Redis服务器时加载配置文件
```sh
# 1.创建文件 D:\ArchitectPracticer\Redis\RedisConf\redis.conf 并设置内容如下
port 6379
bind 127.0.0.1
timeout 300
# port为端口号，bind为绑定的ip,timeout为连上的客户端空闲300s后，服务器将终止该客户端的连接，这些均是默认值
# 上面的路径和文件名都是可以改变的

# 2.使用redis.conf配置文件启动容器
PS D:\code\blogs\farb.github.io> docker run -itd --name redisWithConfig -v D:\ArchitectPracticer\Redis\RedisConf\redis.conf:/redisConfig/redis.conf -p 6379:6379 redis:latest redis-server/redisConfig/redis.conf 
d2944577be148ee004cce434a72db5e5a3d9264d0129130c1e61616fc199a5cf
docker: Error response from daemon: driver failed programming external connectivity on endpoint redisWithConfig (82c4756e055f9861e0804672db4e0b9ac108558b6e7e8e85b3f15af71319e932): Bind for 0.0.0.0:6379 failed: port is already allocated.
# 因为之前已经创建了一个firstRedis容器占用了6379端口，所以先要删除之前的容器
docker rm firstRedis

PS D:\code\blogs\farb.github.io> docker run -itd --name redisWithConfig -v D:\ArchitectPracticer\Redis\RedisConf\redis.conf:/redisConfig/redis.conf -p 6379:6379 redis:latest redis-server /redisConfig/redis.conf 
ca7280d0f6de05f0d26067e21b68ed9a231480f66b051a456662305fcfedda88
PS D:\code\blogs\farb.github.io> docker ps
CONTAINER ID   IMAGE          COMMAND                   CREATED         STATUS         PORTS                    NAMES
ca7280d0f6de   redis:latest   "docker-entrypoint.s…"   3 seconds ago   Up 2 seconds   0.0.0.0:6379->6379/tcp   redisWithConfig
# -itd指定以分离的交互式终端运行，
# --name设置容器的名称
# -v 指定本机和Docker内目录的映射关系，注意先本机路径，再docker路径
# -p 指定docker的6379端口映射到主机的6379端口，注意先docker，再本机
# redis:latest 为最新的redis镜像
# 再用redis-server /redisConfig/redis.conf指定启动容器时默认运行的命令

# 3.验证保存配置
docker exec -it redisWithConfig /bin/bash
root@ca7280d0f6de:/data# redis-cli
127.0.0.1:6379> config get requirepass
1) "requirepass"
2) ""
127.0.0.1:6379> config set requirepass 123456
OK
127.0.0.1:6379> config rewrite
OK

```

### 多个客户端连接远端服务器
#### 多个Redis客户端连接远端服务器
**启动三个容器，一个容器启动Redis服务器，两个容器启动两个Redis客户端，模拟两个客户端远程连接到服务器**
```sh
# 启动一个容器，保护Redis服务器
PS D:\code\blogs\farb.github.io> docker run -itd --name redis-server redis:latest
bbd384d02e177ab6b65ca7f0670707d250b16ec0bf1596c75d0db879ed9e2065

# 启动客户端1，并设置值
# --link redis-server:server 参数项连接到了redis-server服务器上，:server表示服务器的别名
PS D:\code\blogs\farb.github.io> docker run -it --name redis-client1 --link redis-server:server redis:latest /bin/bash
root@10038abf0e0f:/data# redis-cli -h server -p 6379
server:6379> set name farbguo
OK
# 启动客户端2，并读取值
PS D:\code\blogs\farb.github.io> docker run -it --name redis-client2 --link redis-server:server redis:latest /bin/bash
root@b8d623344268:/data# redis-cli -h server -p 6379
server:6379> get name
"farbguo"
```

#### 通过docker inspect观察IP地址
```sh
docker inspect redis-client1
# 在redis-server所在容器中查看客户端1的信息，会返回很长信息，下面仅截取部分。
     "Networks": {
                "bridge": {
                    "IPAMConfig": null,
                    "Links": null,
                    "Aliases": null,
                    "MacAddress": "02:42:ac:11:00:04",
                    "NetworkID": "aa279f1a1aca3000771c18447f8fa16439fc4339cf3522f19bfad3abf7cbbba9",
                    "EndpointID": "f8e8ef3128c0f83b00a89b7ad53d5ee8f3916b50a44b718fbef478ea1b57a895",
                    "Gateway": "172.17.0.1",
                    "IPAddress": "172.17.0.4",
                    "IPPrefixLen": 16,

```

#### 查看所有客户端
```sh
# 在redis-client2容器上查看所有客户端
server:6379> client list
id=4 addr=172.17.0.5:38928 laddr=172.17.0.3:6379 fd=9 name= age=892 idle=0 flags=N db=0 sub=0 psub=0 ssub=0 multi=-1 qbuf=26 qbuf-free=20448 argv-mem=10 multi-mem=0 rbs=1024 rbp=0 obl=0 oll=0 omem=0 tot-mem=22426 events=r cmd=client|list user=default redir=-1 resp=2 lib-name= lib-ver=
id=5 addr=172.17.0.4:59982 laddr=172.17.0.3:6379 fd=8 name= age=5 idle=5 flags=N db=0 sub=0 psub=0 ssub=0 multi=-1 qbuf=0 qbuf-free=0 argv-mem=0 multi-mem=0 rbs=1024 rbp=0 obl=0 oll=0 omem=0 tot-mem=1928 events=r cmd=command|docs user=default redir=-1 resp=2 lib-name= lib-ver=

# 以上发现，id为4的客户端name为空，下面修改之后就有值了。
server:6379> client setname client2
OK
server:6379> client getname
"client2"
server:6379> client list
id=4 addr=172.17.0.5:38928 laddr=172.17.0.3:6379 fd=9 name=client2 age=992 idle=0 flags=N db=0 sub=0 psub=0 ssub=0 multi=-1 qbuf=26 qbuf-free=20448 argv-mem=10 multi-mem=0 rbs=1024 rbp=0 obl=0 oll=0 omem=0 tot-mem=22426 events=r cmd=client|list user=default redir=-1 resp=2 lib-name= lib-ver=
id=5 addr=172.17.0.4:59982 laddr=172.17.0.3:6379 fd=8 name= age=105 idle=105 flags=N db=0 sub=0 psub=0 ssub=0 multi=-1 qbuf=0 qbuf-free=0 argv-mem=0 multi-mem=0 rbs=1024 rbp=0 obl=0 oll=0 omem=0 tot-mem=1928 events=r cmd=command|docs user=default redir=-1 resp=2 lib-name= lib-ver=
```

#### 通过info查看服务器状态
```sh
# 可以在任何一台客户端上查看服务器状态信息，
# 通过info section命令，可以查看每个配置节点的信息
server:6379> info keyspace
# Keyspace
db0:keys=1,expires=0,avg_ttl=0
```

**注意：在任何一个客户端使用shutdown命令都可以关闭服务器和所有连接的客户端。生产环境务必注意，否则可能造成数据丢失，导致事故**


