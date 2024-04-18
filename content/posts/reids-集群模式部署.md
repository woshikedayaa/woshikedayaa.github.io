---
title: "Reids 哨兵集群模式"
date: 2023-12-14T16:54:15+08:00
draft: false
tags:
 - redis
 - docker
---

**开始前先推荐一篇文章**:https://zhuanlan.zhihu.com/p/177000194

# 开始

redis 有三种模式。其中最能实现**高可用[HA(High Avaliable)]且性能好**的莫属于集群模式。下面我使用docker-compose实现一个集群模式

首先准备一个配置文件模板

```yaml
# cluster-conf.conf
port ${PORT} # 端口
protected-mode no # 关闭 保护模式
daemonize no # 关闭 守护线程模式
appendonly yes # 开启 aof日志 推荐看看:https://c.biancheng.net/redis/aof.html
cluster-enabled yes # 开启 集群模式
cluster-config-file nodes.conf # 集群节点信息文件 (有解释)
cluster-node-timeout 15000 # 集群连接超时时间 单位:ms
cluster-announce-ip 172.30.28.83 # 集群节点ip 这里填主机ip (获取ip下文有讲)
cluster-announce-port ${PORT} # 指定 集群节点映射端口
cluster-announce-bus-port 1${PORT} # 指定 集群节点总线端口 (有解释)
bind 0.0.0.0 # 暴露
```

**集群节点总线**

```tex
每个 Redis 集群节点都需要打开两个 TCP 连接。
一个用于为客户端提供服务的正常 Redis TCP 端口，例如 6379。
还有一个基于 6379 端口加 10000 的端口，比如 16379。
第二个端口用于集群总线，这是一个使用二进制协议的节点到节点通信通道。
节点使用集群总线进行故障检测、配置更新、故障转移授权等等。
客户端永远不要尝试与集群总线端口通信，与正常的 Redis 命令端口通信即可。
但是请确保防火墙中的这两个端口都已经打开，否则 Redis 集群节点将无法通信。
👉 这里有坑 详见下文
```

**获取主机ip方法**

```bash
$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:15:5d:99:4f:ac brd ff:ff:ff:ff:ff:ff
    inet 172.30.28.83/20 brd 172.30.31.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::215:5dff:fe99:4fac/64 scope link
       valid_lft forever preferred_lft forever
......

# 1: lo: <LOOPBACK,UP,LOWER_UP> 
# 指的是本地网络回环地址,就是熟悉的 127.0.0.1
# 2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP>
# 这个指的是你的网卡(eth0) 绑定的地址
# 第三行的 inet 172.30.28.83 
# 其中172.30.28.83 就是宿主机(相较于docker)地址
```

**集群节点信息文件**

```tex	
这个文件，用于存储集群的状态信息，包括节点的信息、槽分配等。
在这个文件中，通常包含了集群的配置信息，每个节点的状态以及节点之间的关系。
这样，在集群重启后，节点可以通过读取这个文件来恢复之前的状态，确保集群的一致性。
👇
人话:保存集群信息的
```

# 部署

我们首先将刚才的文件复制6份 并且配置其中的端口号

```bash
$ for i in `seq 1 6`; do\
    cp "cluster-conf.conf" "cluster-conf${i}.conf"&&\
    sed -i "s/\${PORT}/638${i}/g" "cluster-conf${i}.conf"&&\
    echo "生成文件：cluster-conf${i}.conf";\
done
生成文件：cluster-conf1.conf
生成文件：cluster-conf2.conf
生成文件：cluster-conf3.conf
生成文件：cluster-conf4.conf
生成文件：cluster-conf5.conf
生成文件：cluster-conf6.conf
$ touch docker-compose.yaml
```

然后开始写compose文件

```yaml
version: '3.8'

networks:
 redisc: # redis-cluster
  driver: bridge

volumes:
 redisc-data1: 
  driver: local
 redisc-data2:
  driver: local
 redisc-data3:
  driver: local
 redisc-data4:
  driver: local
 redisc-data5:
  driver: local
 redisc-data6:
  driver: local

services:
 redis1:
  image: redis # 镜像
  container_name: redis1 # 容器名称
  command: redis-server /etc/redis/cluster-conf.conf # 修改启动命令
  volumes:
   - redisc-data1:/data # data目录 (持久化相关)
   - ./cluster-conf1.conf:/etc/redis/cluster-conf.conf # 配置文件
  ports:
   - "6381:6381"
   - "16381:16381"
  networks:
   - redisc

 redis2:
  image: redis
  container_name: redis2
  command: redis-server /etc/redis/cluster-conf.conf
  volumes:
   - redisc-data2:/data
   - ./cluster-conf2.conf:/etc/redis/cluster-conf.conf
  ports:
   - "6382:6382"
   - "16382:16382"
  networks:
   - redisc

 redis3:
  image: redis
  container_name: redis3
  command: redis-server /etc/redis/cluster-conf.conf
  volumes:
   - redisc-data3:/data
   - ./cluster-conf3.conf:/etc/redis/cluster-conf.conf
  ports:
   - "6383:6383"
   - "16383:16383"
  networks:
   - redisc

 redis4:
  image: redis
  container_name: redis4
  command: redis-server /etc/redis/cluster-conf.conf
  volumes:
   - redisc-data4:/data
   - ./cluster-conf4.conf:/etc/redis/cluster-conf.conf
  ports:
   - "6384:6384"
   - "16384:16384"
  networks:
   - redisc

 redis5:
  image: redis
  container_name: redis5
  command: redis-server /etc/redis/cluster-conf.conf
  volumes:
   - redisc-data5:/data
   - ./cluster-conf5.conf:/etc/redis/cluster-conf.conf
  ports:
   - "6385:6385"
   - "16385:16385"
  networks:
   - redisc

 redis6:
  image: redis
  container_name: redis6
  command: redis-server /etc/redis/cluster-conf.conf
  volumes:
   - redisc-data6:/data
   - ./cluster-conf6.conf:/etc/redis/cluster-conf.conf
  ports:
   - "6386:6386"
   - "16386:16386"
  networks:
   - redisc 
```

启动容器

```bash
$ docker compose up -d
```

启动过后 容器应该是正常运行的 但是这个时候还没有形成集群

我们需要进入随便一个容器配置集群

这里我就以第6个容器为例

```bash
$ docker exec -it redis6 bash
$ cd /usr/local/bin/
$ ls 
docker-entrypoint.sh  gosu  redis-benchmark  redis-check-aof  redis-check-rdb  redis-cli  redis-sentinel  redis-server
# 可以看到这里基本都是关于redis的文件
# 先测试redis的连接 -c 代表了以集群模式连接
$ redis-cli -h 172.30.28.83 -p 6381 -c 
172.30.28.83:6381>cluster info
cluster_state:fail 
# 可以看到这里集群状态是 fail 代表了集群并没有形成 下面开始手动配置
...
172.30.28.83:6381>exit
$ redis-cli --cluster create \
172.30.28.83:6381 172.30.28.83:6382 172.30.28.83:6383 \
172.30.28.83:6384 172.30.28.83:6385 172.30.28.83:6386 \
--cluster-replicas 1
# 上面代表了创建集群 请把ip换成自己的ip
# 中途会确认一次 直接输入 yes 即可
# 创建完成后再来测试一次
$ redis-cli -h 172.30.28.83 -p 6381 -c
172.30.28.83:6381>cluster info
cluster_state:ok
# ok 代表集群配置成功了
...
```

# 踩坑总结

1.一直卡在 **Waiting for the cluster to join**

其实与配置文件中的 **cluster-announce-bus-port** 有关

这里摘选官方文档一句话

```tex
Redis集群中的各个节点，需要开放一个端口，同其他节点建立连接，用于接收心跳数据等操作。也就是说，redis-node1节点，开放6379端口供client连接时，同时提供16379端口(10000 + 6379)，供其他Redis节点连接。
```

集群初始化过程中，需要同其他Redis建立连接，进行通信。若节点间无法连接，此时会阻塞，这也就是一直阻塞到"Waiting for the cluster to join"环节的原因。

如果你按我的步骤来 这个大概是因为宿主机防火墙没配置

这里以ubuntu 举例 (其他系统自行寻找方法)

```bash
$ sudo systemctl disable ufw
```

这样是关闭防火墙了 当然你也可以配置防火墙对应端口

当然作者在部署的时候是踩坑了的 偷懒的原因 下文详细解释

