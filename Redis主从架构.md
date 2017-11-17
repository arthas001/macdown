# Redis 主从架构

本文主要说明采用一主多从架构方案的搭建，并使用 `Redis Sentinel` 做监控故障迁移(master机器挂掉后会从slave机器中选出一个作为master机器)。此主从架构在业务量增长较多后会有一定局限性，可以根据实际情况考虑增加内存或者考虑分片集群。

此主从分布架构图：
![Redis与Sentinel架构](/Users/hxp/Desktop/WechatIMG11.jpeg)

下面的操作在单机器中进行，实际生产环境要在多机器中部署较好。
## 安装 redis
```bash
wget http://download.redis.io/releases/redis-4.0.2.tar.gz && \
tar xzf redis-4.0.2.tar.gz && \
cd redis-4.0.2 && \
make && \
cd src

# cp redis-* 文件到 /usr/local/bin目录
sudo cp redis-server /usr/local/bin/ && \
sudo cp redis-benchmark /usr/local/bin/ && \
sudo cp redis-check-aof /usr/local/bin/ && \
sudo cp redis-check-rdb /usr/local/bin/ && \
sudo cp redis-cli /usr/local/bin/ && \
sudo cp redis-sentinel /usr/local/bin
```


## 配置redis.conf, 启动一主二从机器
创建master配置`redis-master.conf`

```shell
tcp-keepalive 60
#bind 127.0.0.1
port 6379
requirepass myredis # 密码
maxmemory-policy noeviction
appendonly yes
appendfilename redis-staging-ao.aof
save 300 10
dbfilename dump.rdb
dir ./
daemonize yes
protected-mode no
```

创建slave-one配置`redis-slave-one.conf`

```bash
tcp-keepalive 60
#bind 127.0.0.1
port 6380
maxmemory-policy noeviction
slaveof 127.0.0.1 6379
masterauth myredis
daemonize yes 
protected-mode no
```

创建slave-two配置`redis-slave-two.conf`

```bash
tcp-keepalive 60
#bind 127.0.0.1
port 6381
maxmemory-policy noeviction
slaveof 127.0.0.1 6379
masterauth myredis
daemonize yes 
protected-mode no
```

启动redis服务

```bash
redis-server redis-master.conf && \
redis-server redis-slave-one.conf && \
redis-server redis-slave-two.conf
```


## 安装 sentinel
需要至少启动3个`sentinel`实例
sentinel配置信息redis-sentinel-1.conf

```bash
bind 127.0.0.1
port 5000
daemonize yes
sentinel monitor mymaster 127.0.0.1 6379 2
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 60000
sentinel parallel-syncs mymaster 1
sentinel auth-pass mymaster myredis # master 密码
```
其他两个配置信息相同，不同的是端口分别为`5002`、`5003`
启动sentinel服务

```bash
redis-sentinel redis-sentinel-1.conf && \
redis-sentinel redis-sentinel-2.conf && \
redis-sentinel redis-sentinel-3.conf
```

## 测试故障转移
连接sentinel服务
`redis-cli -p 5000`
首先查看当前的master服务信息

```bash
127.0.0.1:5000> SENTINEL get-master-addr-by-name mymaster
1) "127.0.0.1"
2) "6379"
```
停止`6379`机器的服务
`redis-cli -p 6379 DEBUG sleep 30`

再次查看当前master服务信息,master已转移至6380端口服务

```bash
127.0.0.1:5000> SENTINEL get-master-addr-by-name mymaster
1) "127.0.0.1"
2) "6380"
```

## 连接
到此基本的主从结构已经搭建完毕，可以连接`6379`端口进行写操作，连接`6380`或`6381`进行读操作。

