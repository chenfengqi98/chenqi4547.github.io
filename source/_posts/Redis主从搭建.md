---
title: Redis主从搭建
date: 2020-04-08 18:26:08
tags:
- Redis
categories:
- Redis
---

### 配置文件准备

先复制三个redis.conf和三个sentinel.conf文件，如图所示。

{% asset_img conf.png %}

---

### 修改配置文件

redis.conf文件主要修改如下几个配置，改成对应的端口号。

```sh
port 6379
pidfile "/var/run/redis_6379.pid"
dbfilename "dump_6379.rdb"
appendfilename "appendonly_6379.aof"
```

sentinel.conf文件修改如下几个配置。

```sh
port 26379
daemonize yes
pidfile "/var/run/redis-sentinel_26379.pid"
# 6379 为master 当两个以上的哨兵认为master主观下线时进行故障转移
sentinel monitor mymaster 127.0.0.1 6379 2
```

然后分别启动三个redis进程和sentinel进程

---

### 启动

先进入到redis安装目录

```shell
#启动三个redis进程
./src/redis-server conf/redis-6379.conf
./src/redis-server conf/redis-6380.conf
./src/redis-server conf/redis-6381.conf

#进入到6379和6380的客户端，并设置为6381的从服务
./src/redis-cli -h 127.0.0.1 -p 6379
replicaof 127.0.0.1 6379

./src/redis-cli -h 127.0.0.1 -p 6380
replicaof 127.0.0.1 6379

#启动三个sentinel进程
./src/redis-server conf/sentinel-26379.conf --sentinel
./src/redis-server conf/sentinel-26380.conf --sentinel
./src/redis-server conf/sentinel-26381.conf --sentinel
```

如图一共有六个进程redis有关的进程

{% asset_img result-1.png %}

---

### 测试

查看主从信息，如图所示。

{% asset_img result-3.png %}

当shutdown一个redis后，是否会重新选举master

{% asset_img result-4.png %}

如图所示，当`shutdown`主服务器后，进行了重新选举master

{% asset_img result-5.png %}

并且配置文件会自动修改`sentinel`监控的主服务，如图所示监控的master变成了6381端口的redis服务

{% asset_img result-6.png %}

---

### Redis主从复制流程

1. 从服务器通过`psync`命令发生服务器已有的同步进度（同步源ID、同步进度offset）。
2. master收到请求，同步源为当前master，则根据偏移量增量同步。
3. 同步源非当前master，则进入全量同步：master生成rdb文件，传输到slave，加载的slave内存。

### 主从复制应用场景

1. 主从复制可以用来支持读写分离，主服务器用来写，从服务器只读。
2. 可以使用主从复制来避免master持久化造成的开销。master关闭持久化，slave配置持久化。但是重新启动的master将总空数据集开始，如果slave试图同步，那么slave也会被清空。

### 主从复制注意事项

1. 读写分离场景

   数据复制延时导致读到过期数据或者读不到数据（网络问题、slave阻塞）。从节点故障，多个client如何迁移。

2. 全量复制

   第一次建立主从关系或者runid不匹配会导致全量复制，故障转移时也会全量复制。

3. 复制风暴

   master故障，如果slave节点很多，所以slave都要复制，对服务器和网络的压力都有影响。

4. 写能力有限

   主从复制只有一个master时写能力有限

5. master故障情况下

   如果master不开启持久化，slave开启持久化，当redis配置了自动重启，将会清空数据（使用sentinel故障转移、主从切换）。

6. 带有效期的key

   slave不会让key过期，而是等待master让key过期，在lua脚本执行期间不执行任何key过期操作。

### 主观下线和客观下线

#### 主观下线

单个哨兵自身认为redis实例已经不能提供服务。

检测机制：哨兵向redis发生ping请求，+PONG，-LOADING，-MASTERDOWN视为正常，其他回复视为异常，可通过`sentinel down-after-milliseconds mymaster 2000`配置时间。

#### 客观下线

一定数量的哨兵认为master已下线。

检测机制：当哨兵主观认为master下线后，会通过`SENTINEL is-master-down-by-addr`命令轮询其他哨兵是否认为master已经下线，如果认为下线的`sentinel`个数超过配置的数量，就会认为master节点客观下线，开启故障转移流程。通过`sentinel monitor mymaster 127.0.0.1 6379 2`配置。

---

### 哨兵如何知道redis主从信息的？

{% asset_img result-7.png %}

哨兵通过`info replication`获取主从信息，master中有所用从服务器的信息。

---

### 哨兵之间如何通信

{% asset_img result-10.png %}

redis实例中会存在`__sentinel__:hello`的频道，可通过`pubsub channels`查看当前redis所以通道信息。{% asset_img result-9.png %}

如图所示，在频道中，三个哨兵都在互相发送消息。

### 哨兵领导选举机制

哨兵选举基于Raft算法实现，流程如下：

1. 拉票阶段：每个哨兵节点希望自己成为leader。
2. sentinel节点收到拉票命令后，如果没有收到或者同意过其他sentinel节点的请求，就同意该节点的请求，每个节点只有一票。
3. 如果sentine节点发现自己票数超过一半，那么它成为leader，执行故障转移。
4. 投票结束后，如果超过`failover-timeout`的时间，没有进行实际的故障转移，则重新拉票选举。

Raft算法：

- https://raft.github.io。

- http://thesecretlivesofdata.com/

---

### slave选举方案

1. slave节点状态

   非S_DOWN，O_DOWN，DISCONNECTED状态。

2. 优先级

   配置文件中`slave-priority`值越小，优先级越高。

3. 数据同步情况

   Replication offset processed

4. 最小的run id

   runid字段顺序

### 主从切换过程

针对即将成为master的slave节点，将其撤出主从集群，自动执行`slaveof no one`。

针对其他slave节点，使它们成为新master的从属，自动执行`slave of host port`。

### 为什么要三个哨兵才能保证健壮性

两个哨兵的情况

{% asset_img result-11.png %}

master宕机了 s1和s2两个哨兵只要有一个认为你宕机了就切换了，并且会选举出一个哨兵去执行故障，但是这个时候也需要大多数哨兵都是运行的。

那这样有啥问题呢？M1宕机了，S1没挂那其实是OK的，但是整个机器都挂了呢？哨兵就只剩下S2个裸屌了，没有哨兵去允许故障转移了，虽然另外一个机器上还有R1，但是故障转移就是不执行。

---