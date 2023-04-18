---
title: Redis持久化与集群配置
categories:
  - Redis
tags:
  - Redis
  - AOF
  - RDB
  - 哨兵模式
  - 主从模式
  - 消息订阅
date: 2023-04-16 15:40:28
---

<!-- toc -->

# 实验环境

**服务器环境**

- Ubuntu 20.04.4
- Redis 7.0.9 

**服务器ip**

Master032004134: 192.168.50.98

Slave032004134: 192.168.50.99

# 实验步骤

## 一、Redis持久化

开启RDB和AOF

按照以下配置修改配置文件：

```
# file: redis.conf
# open AOF
appendonly yes
```

存入以下键值

```shell
# redis-cli
set SID 032004134
set Time 202304141941
```

使用命令，强制Redis保存RDB和AOF

```shell
# redis-cli
BGSAVE # 强制通过异步方式保存RDB
BGREWRITEAOF # 重写AOF
```

将刚刚电脑A上Redis服务器保存的RDB和AOF文件拷入电脑B，配置电脑B上Redis服务器的conf文件，使得在电脑B上的Redis服务器可以通过电脑A上Redis服务器的AOF和RDB文件恢复数据（做两次，先只拷入电脑A的AOF文件在电脑B上恢复一次数据；接着删除B上的AOF文件，尝试用电脑A的RDB文件再在电脑B上使用RDB文件恢复一次）

同步文件`rsync -av redis-stable Slave032004134:/usr/local/`

在拥有AOF和RDB的状态下启动Redis

删除aof目录下的增量aof文件，并且修改**appendonly.aof.manifest**文件，令其保留第一行

在只有RDB的状态下启动Redis

查看电脑B上Redis服务器此时的SID与Time的值是否与电脑A的Redis服务器的值一致

```shell
# redis-cli
get SID
get Time
```

## 二、Redis主从复制

其中，关闭master的RDB和AOF，并开启slave1,slave2,slave3的RDB和AOF（注意几个服务器的AOF和RDB的文件名**不要重名**了），从服务器均为**只读**

集群信息:

- Master：Master032004134:6379
- Slave1：Slave032004134:6380
- Slave2：Slave032004134:6381
- Slave3：Slave032004134:6382

配置Master服务器上的配置文件

```shell
# Master032004134:3079
# 正常在集群中不应该关闭主节点的AOF和RDB
appendonly no
save ""
# 关闭保护模式，以允许非本地连接
protected-mode no
# 关闭bind，这样子可以监听所有的请求
即注释掉 bind Slave032004134 -::1
```

配置Slave服务器上的配置文件

```shell
# 通用配置
replicaof Master032004134 6379
replica-read-only yes
# slave独立配置，替换端口就好
# 使用:%s/old/new/gc进行批量替换
port 6382
pidfile /var/run/redis_6382.pid
logfile "/usr/local/redis-stable/log/log_6382.txt"
dbfilename dump_6382.rdb
appendfilename "appendonly_6382.aof"
```

启动主节点，启动从节点

在6379服务器设置键值site，值为www.baidu.com，观察6380、6381、6382是否可以读取到site

在master的redis-cli输入：`set site www.baidu.com`

观察6380、6381、6382是否可以读取到site

假定Master服务器突然宕机（直接用shutdown关闭），手动输入命令将127.0.0.1:6380设为新的Master服务器:6381和6382改为127.0.0.1:6380的从服务器

执行以下命令

```
# Redis-Cli
# 在6381和6382上
REPLICAOF Slave032004134 6380
# 查看主从状态
INFO replication
```

## 三、Redis哨兵模式的启用

重新恢复上述服务器之间的主从关系

给6379和6381都设置登录密码：passwd，配置6380、6381和6382服务器使其仍能与6379同步

配置Sentinel哨兵模式监视Master（只需要一个哨兵），并且配置服务器的优先级，使得Master宕机后由127.0.0.1:6381接替Master，127.0.0.1:6380和6382改为127.0.0.1:6381的从服务器，配置完成后，启动哨兵模式，关闭6379服务器，观察整个过程是否成功

```shell
# 配置6379 sentinel.conf
daemonize yes
logfile "/usr/local/redis-stable/log/log-sentinel.txt"
sentinel monitor mymaster 192.168.50.98 6379 1
sentinel auth-pass mymaster 3079
sentinel down-after-milliseconds mymaster 1000
sentinel failover-timeout mymaster 10000
sentinel parallel-syncs mymaster 1
# 配置6379
requirepass 3079
masterauth 3079
# 配置6380
masterauth 3079
注释掉bind
关闭保护模式
# 配置6381
requirepass 3079
masterauth 3079
replica-priority 10
注释掉bind
关闭保护模式
# 配置6382 
masterauth 3079
注释掉bind
关闭保护模式
```

## 四、关系型数据库中表在Redis中的设计与存储

根据文档“redis中关系型表的存储规范”介绍的方法将下表以键值形式存储：

Key名: “表名:主键名:主键的值:列名”  

Value:  列值

| ID   | Name     | English | Math |
| ---- | -------- | ------- | ---- |
| 1    | zhangsan | 69      | 86   |
| 2    | Lisi     | 69      | 100  |

```shell
# redis-cli
set student:ID:1:Name zhangsan
set student:ID:1:English 69
set student:ID:1:Math 86

set student:ID:2:Name Lisi
set student:ID:2:English 69
set student:ID:2:Math 100
# 冗余存储
sadd student:Name:zhangsan:ID 1
sadd student:English:69:ID 1
sadd student:Math:86:ID 1

sadd student:Name:Lisi:ID 2
sadd student:English:69:ID 2
sadd student:Math:100:ID 2
```

在redis数据库执行以下查找

```shell
# 查找数学成绩为86的学生ID
smembers student:Math:86:ID
# 查找数学成绩为100的学生姓名
smembers student:Math:100:ID
get student:ID:2:Name
# 查找英语成绩为69且数学成绩为100的学生姓名
sinter student:English:69:ID student:Math:100:ID
get student:ID:2:Name
# 查找英语成绩为69或者数学成绩为86的学生姓名
sunion student:English:69:ID student:Math:86:ID
get student:ID:2:Name
get student:ID:1:Name
```

## 五、Redis消息功能

1、开启三个客户端，A客户端负责在频道发送各个城市的天气信息，频道名分别为City_Beijing、City_Shanghai和City_Guangzhou，B和C客户端负责接收频道信息

执行以下命令

```shell
# 在B客户端输入命令令其订阅北京的天气信息
subscribe City_Beijing
# 使用通配符，让C客户端接收所有城市的天气信息
Psubscribe City_*
# A客户端，在City_Beijing频道发送sunny，在City_Shanghai频道发送cloudy，在City_Guangzhou发送rainy，观察客户端B，C接收情况
publish City_Beijing sunny
publish City_Shanghai cloudy
publish City_Guangzhou rainy
```



# 遇到的问题

1. redis 没有按照配置文件的配置运行在后台

   要在命令后面加上配置文件，redis服务器才会像配置文件中的配置一样运行。

2. Error condition on socket for SYNC: Connection refused

   1. **mode**=standlone,没有开启Redis的集群模式
   2. **bind**设置为监听127.0.0.1的所有端口，所以监听端口只有监听本地的端口，不能监听到从服务器的端口
   3. master处于**protected**模式，只能接收本地的连接

3. redis-cli报错`CLUSTERDOWN Hash slot not served`

   通常表示 Redis 集群中的某些节点不可用或当前节点与其他节点之间的网络连接发生了故障。

   因为我的集群只有一个主节点，但是Redis集群至少要求要3个主节点

4. redis配置密码却没有生效

   配置文件最后一行`user default on nopass ~* &* +@all`导致

5. 哨兵模式无法选择从节点作为新的主节点

   因为主从节点在不同的服务器上，从节点没有配置可以监听其他服务器的请求，所以哨兵无法连接到从节点，导致无法选择从节点作为新的主节点

6. 开启哨兵模式后，从节点挂了

   因为哨兵模式开启后，回去修改从节点的配置文件，而本来的主节点的ip设置的是127.0.0.1，然后导致从节点的配置中，主节点的ip变成了127.0.0.1，但主从节点并没有在同一台服务器内，所以无法同信。

# 参考

1. https://redis.io/docs/management/replication
2. https://redis.io/docs/management/persistence
2. [(97条消息) Redis三种集群方式安装及配置_redis集群配置_Rouge丶铠的博客-CSDN博客](https://blog.csdn.net/RougeK/article/details/108815028)
