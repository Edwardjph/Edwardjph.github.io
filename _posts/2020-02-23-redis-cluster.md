---
title: 搭建redis cluster
tags: redis
---

### 一、安装redis（[搭建redis sentinel](https://edwardjph.github.io/2020/02/22/redis-sentinel.html)）

---

### 二、搭建redis cluster（[官网教程](https://redis.io/topics/cluster-tutorial)）

按预期工作的最小群集要求至少包含三个主节点。对于您的第一个测试，强烈建议启动一个包含三个主节点和三个从节点的六个节点群集。

##### 在redis-5.0.7目录下创建redis-cluster，并创建六个目录以端口号命名

```shell
mkdir redis-cluster
cd redis-cluster
mkdir 7000 7001 7002 7003 7004 7005
```

##### 复制配置文件

```shell
cp redis-5.0.7/redis.conf redis-5.0.7/redis-cluster/7000/redis.conf
cp redis-5.0.7/redis.conf redis-5.0.7/redis-cluster/7001/redis.conf
cp redis-5.0.7/redis.conf redis-5.0.7/redis-cluster/7002/redis.conf
cp redis-5.0.7/redis.conf redis-5.0.7/redis-cluster/7003/redis.conf
cp redis-5.0.7/redis.conf redis-5.0.7/redis-cluster/7004/redis.conf
cp redis-5.0.7/redis.conf redis-5.0.7/redis-cluster/7005/redis.conf
```

##### 修改配置文件

```shell
port 7000
cluster-enabled yes #启用集群模式
cluster-config-file nodes-7000.conf #集群配置文件，启动时自动生成
cluster-node-timeout 5000 #超时时间
appendonly yes #在每次更新操作后进行日志记录，Redis在默认情况下是异步的把数据写入磁盘，如果不开启，可能会在断电时导致一段时间内的数据丢失
#后台启动，配置如下
daemonize yes 
pidfile  /var/run/redis_7001.pid
#如需外网访问，配置如下
protected-mode no
bing 0.0.0.0
#如需设置密码，配置如下
masterauth 123456
requirepass 123456
```

其中 port、cluster-config-file、pidfile 需要随着 文件夹的不同调增

##### 启动节点

```shell
redis-5.0.7/src/redis-server redis-5.0.7/redis-cluster/7000/redis.conf
redis-5.0.7/src/redis-server redis-5.0.7/redis-cluster/7001/redis.conf
redis-5.0.7/src/redis-server redis-5.0.7/redis-cluster/7002/redis.conf
redis-5.0.7/src/redis-server redis-5.0.7/redis-cluster/7003/redis.conf
redis-5.0.7/src/redis-server redis-5.0.7/redis-cluster/7004/redis.conf
redis-5.0.7/src/redis-server redis-5.0.7/redis-cluster/7005/redis.conf
```

##### 启动集群

```shell
redis-cli --cluster create 127.0.0.1:7000 127.0.0.1:7001 \
127.0.0.1:7002 127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005 \
--cluster-replicas 1 #为每个创建的主机都提供一个从机
```

如果成功将看到

```shell
[OK] All 16384 slots covered
```

#### 使用create-cluster脚本创建Redis集群

##### 修改官方提供的脚本

```shell
vi redis-5.0.7/utils/create-cluster/create-cluster 

#!/bin/bash

# Settings
PORT=6999
TIMEOUT=2000
NODES=6
REPLICAS=1
```

端口设置为6999，NODES为6，工具会自动累加1 生成 7000-7005 六个节点 用于操作

##### 启动节点

```shell
[root@localhost create-cluster]# sh create-cluster start
```

**注：必须位于此目录下，脚本中有“../../src/redis-server”语句**

##### 启动集群

```shell
[root@localhost create-cluster]# sh create-cluster create
```

##### 停止集群

```shell
[root@localhost create-cluster]# sh create-cluster stop
```

### 三、测试及操作集群

```shell
$ redis-cli -c -p 7000
redis 127.0.0.1:7000> set foo bar
-> Redirected to slot [12182] located at 127.0.0.1:7002
OK
redis 127.0.0.1:7002> set hello world
-> Redirected to slot [866] located at 127.0.0.1:7000
OK
redis 127.0.0.1:7000> get foo
-> Redirected to slot [12182] located at 127.0.0.1:7002
"bar"
redis 127.0.0.1:7000> get hello
-> Redirected to slot [866] located at 127.0.0.1:7000
"world"
```

#### 重新分簇

```shell
redis-cli --cluster reshard 127.0.0.1:7000
```

填写对应信息

```
How many slots do you want to move (from 1 to 16384)?
What is the receiving node ID? 
```

node ID已由redis-cli打印在列表中，也可通过

```shell
redis-cli -p 7000 cluster nodes | grep myself
```

操作成功后查看集群信息

```shell
redis-cli --cluster check 127.0.0.1:7000
```

自动执行命令

```shell
redis-cli reshard <host>:<port> --cluster-from <node-id> --cluster-to <node-id> --cluster-slots <number of slots> --cluster-yes
```

#### 添加一个新节点

添加一个新节点7006，除了端口号外，与其他节点使用相同的配置，按之前节点使用的设置顺序进行操作：

- 创建一个名为的目录`7006`
- 在内部创建一个redis.conf文件，类似于用于其他节点的文件，但使用7006作为端口号
- 启动服务器 

此时服务器应该正在运行

添加节点

```shell
redis-cli --cluster add-node 127.0.0.1:7006 127.0.0.1:7000
```

将新节点的地址指定为第一个参数，并将集群中随机存在的节点的地址指定为第二个参数，此时

- 由于没有分配的哈希槽，因此不保存任何数据。
- 因为它是没有分配插槽的主机，所以当从机要成为主机时，它不会参与选举过程。

现在可以使用的重新分片功能为此节点分配哈希槽

也可以将新节点设置为副本

```shell
cluster replicate 3c3a0c74aae0b56170ccb03a76b60cfe7dc1912e
```

将新节点设置为副本的便捷方式

- 将新节点添加为副本较少的主节点中的随机主节点的副本

  ```shell
  redis-cli --cluster add-node 127.0.0.1:7006 127.0.0.1:7000 --cluster-slave
  ```

- 将新副本分配给特定的主数据库

  ```shell
  redis-cli --cluster add-node 127.0.0.1:7006 127.0.0.1:7000 --cluster-slave --cluster-master-id 3c3a0c74aae0b56170ccb03a76b60cfe7dc1912e
  ```

#### 删除节点

```shell
redis-cli --cluster del-node 127.0.0.1:7000 `<node-id>`
```

第一个参数只是集群中的一个随机节点，第二个参数是您要删除的节点的ID

**但是要删除主节点，它必须为空**