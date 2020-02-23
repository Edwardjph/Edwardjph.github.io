---
title: 搭建redis sentinel
tags: redis
---

### 一、安装redis

[官网下载链接及安装教程](https://redis.io/download)

```shell
$ wget http://download.redis.io/releases/redis-5.0.7.tar.gz
$ tar xzf redis-5.0.7.tar.gz
$ cd redis-5.0.7
$ make
```

**注：make时报错，没有gcc，需安装gcc，再次安装会提示：jemalloc/jemalloc.h：没有那个文件或目录**

**是因为上次编译失败，有残留的文件，需执行：**

```shell
make distclean
make
```

### 二、搭建redis sentinel（[官网搭建教程](https://redis.io/topics/sentinel)）

#### 准备三个redis服务

复制配置文件

```shell
cp redis-5.0.7/redis.conf redis-5.0.7/redis-sentinel/redis-16379.conf
cp redis-5.0.7/redis.conf redis-5.0.7/redis-sentinel/redis-26379.conf
cp redis-5.0.7/redis.conf redis-5.0.7/redis-sentinel/redis-36379.conf
```

修改配置文件

- 主节点：16379

  ```shell
  daemonize yes
  pidfile /var/run/redis-16379.pid
  logfile /var/log/redis/redis-16379.log
  port 16379
  bind 0.0.0.0
  protected-mode no
  dbfilename dump-16379.db
  ```

  **daemonize yes：使用yes启用守护进程（后台启动，方便单机操作）**

  **pidfile：当Redis以守护进程方式运行时，Redis默认会把pid写入/var/run/redis.pid文件，可以通过pidfile指定**

  **bind：默认的接口是127.0.0.1，也就是本地回环地址。这样的话，访问redis服务只能通过本机的客户端连接，而无法通过远程连接。**

  **protected-mode：为了禁止外网访问redis，如果启用了，则只能够通过lookback ip（127.0.0.1）访问Redis**

  **dbfilename：指定本地数据库文件名，默认值为dump.rdb**

- 从节点：redis-26379

  ```shell
  daemonize yes
  pidfile /var/run/redis-26379.pid
  logfile /var/log/redis/redis-26379.log
  port 26379
  bind 0.0.0.0
  protected-mode no
  dbfilename dump-26379.db
  slaveof 127.0.0.1 16379
  ```

  **slaveof：设置本机为slave服务时，设置master服务的IP地址及端口，在Redis启动时，它会自动从master进行数据同步**

- 从节点：redis-36379

  ```shell
  daemonize yes
  pidfile /var/run/redis-36379.pid
  logfile /var/log/redis/redis-36379.log
  port 36379
  bind 0.0.0.0
  protected-mode no
  dbfilename dump-36379.db
  slaveof 127.0.0.1 16379
  ```

启动三个redis服务

```shell
redis-5.0.7/src/redis-server redis-5.0.7/redis-sentinel/redis-16379.conf 
redis-5.0.7/src/redis-server redis-5.0.7/redis-sentinel/redis-26379.conf 
redis-5.0.7/src/redis-server redis-5.0.7/redis-sentinel/redis-36379.conf 
```

查看启动进程：

```shell
[root@localhost ~]# ps -ef|grep redis-server
root       9219      1  0 18:44 ?        00:00:21 src/redis-server 127.0.0.1:16379
root      29754      1  0 19:57 ?        00:00:13 src/redis-server 127.0.0.1:26379
root      30840      1  0 20:01 ?        00:00:11 src/redis-server 127.0.0.1:36379
root      53249  24602  0 21:33 pts/0    00:00:00 grep --color=auto redis-server
```

#### 搭建三个sentinel

复制配置文件

```
cp redis-5.0.7/sentinel.conf redis-5.0.7/redis-sentinel/sentinel-16380.conf
cp redis-5.0.7/sentinel.conf redis-5.0.7/redis-sentinel/sentinel-26380.conf
cp redis-5.0.7/sentinel.conf redis-5.0.7/redis-sentinel/sentinel-36380.conf
```

修改配置文件

```shell
protected-mode no
port 16380
daemonize yes
sentinel monitor master 127.0.0.1 16379 2
sentinel down-after-milliseconds master 5000
sentinel failover-timeout master 180000
sentinel parallel-syncs master 1
logfile /var/log/redis/sentinel-16380.log
```

**sentinel monitor master：指定哨兵监控的主机ip 端口 仲裁数（当有超过仲裁数的哨兵认为master失联，则master客观下线）**

**sentinel down-after-milliseconds master：指定主节点应答哨兵的最大时间间隔，超过这个时间，哨兵主观上认为主节点下线，默认30秒**

**sentinel failover-timeout master：故障转移的超时时间，默认三分钟**

**sentinel parallel-syncs master：指定了在发生failover主备切换时，最多可以有多少个slave同时对新的master进行同步。这个数字越小，完成failover所需的时间就越长；反之，但是如果这个数字越大，就意味着越多的slave因为replication而不可用。可以通过将这个值设为1，来保证每次只有一个slave，处于不能处理命令请求的状态。**

另外两个哨兵节点配置相同，修改对应端口号

启动Sentinel

```shell
redis-5.0.7/src/redis-sentinel redis-5.0.7/redis-sentinel/sentinel-16380.conf
redis-5.0.7/src/redis-sentinel redis-5.0.7/redis-sentinel/sentinel-26380.conf
redis-5.0.7/src/redis-sentinel redis-5.0.7/redis-sentinel/sentinel-36380.conf
```

查看Sentinel启动进程

```shell
[root@localhost ~]# ps -ef|grep redis-sentinel
root      12005      1  0 18:55 ?        00:00:40 src/redis-sentinel *:16380 [sentinel]
root      12027      1  0 18:56 ?        00:00:40 src/redis-sentinel *:26380 [sentinel]
root      12047      1  0 18:56 ?        00:00:40 src/redis-sentinel *:36380 [sentinel]
root      72436  24602  0 22:51 pts/0    00:00:00 grep --color=auto redis-sentinel
```

成功启动后，Sentinel配置将刷新

```shell
sentinel myid 81ba11d4fb90dd463c505994f27297dd127655c8
sentinel config-epoch mymaster 0
sentinel leader-epoch mymaster 0
sentinel known-replica mymaster 127.0.0.1 36379
sentinel known-replica mymaster 127.0.0.1 26379
sentinel known-sentinel mymaster 127.0.0.1 36380 4a43e008998688742b635dc40bb20d1ffd1e7905
sentinel known-sentinel mymaster 127.0.0.1 26380 12d244bb6b1d59faf781df8523b89b76b3d51868
sentinel current-epoch 0
```

#### Sentinel客户端命令

检查状态，返回pong为正常

```bash
>PING
```

显示被监控的所有master以及它们的状态

```bash
>SENTINEL masters
```

显示指定master的信息和状态

```bash
>SENTINEL master <master_name>
```

显示指定master的所有slave以及它们的状态

```shell
>SENTINEL slaves <master_name>
```

显示指定master的所有sentinel以及它们的状态

```shell
>SENTINEL sentinels <master_name>
```

获取当前master的地址

```shell
>SENTINEL get-master-addr-by-name mymaster
```

强制当前sentinel执行failover，不需要其他sentinel节点的同意

```
>SENTINEL failover <master_name>
```

#### 测试故障转移

查看redis-server进程

```shell
[root@localhost ~]# ps -ef|grep redis-server
root       9219      1  0 18:44 ?        00:00:21 src/redis-server 127.0.0.1:16379
root      29754      1  0 19:57 ?        00:00:13 src/redis-server 127.0.0.1:26379
root      30840      1  0 20:01 ?        00:00:11 src/redis-server 127.0.0.1:36379
root      53249  24602  0 21:33 pts/0    00:00:00 grep --color=auto redis-server
```

杀死master

```shell
kill -9 9219
```

链接sentinel-16380,查看master信息

```shell
redis-5.0.7/src/redis-cli -p 16380
127.0.0.1:16380>SENTINEL get-master-addr-by-name mymaster
1) "127.0.0.1"
2) "36379"
```

可以看到主节点已经变为36379

重启redis-server16379

查看配置可以发现成为36379的从节点

