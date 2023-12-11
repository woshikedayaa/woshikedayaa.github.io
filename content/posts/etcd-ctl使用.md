---
title: "Etcd Ctl使用"
date: 2023-12-11T17:28:15+08:00
draft: true
---

# 介绍

etcdctl 是官方提供的操作etcd的一个命令行工具

## 安装

官方发布地址:https://github.com/etcd-io/etcd/releases

找到对应版本 例如我这里是 linux-amd64

```bash
$ wget https://github.com/etcd-io/etcd/releases/download/v3.5.11/etcd-v3.5.11-linux-amd64.tar.gz 
...
$ tar -zxvf etcd-v3.5.11-linux-amd64.tar.gz
# 添加到path中去
$ cp etcd-v3.5.11-linux-amd64/etcdctl etcd-v3.5.11-linux-amd64/etcdutl /usr/local/bin
# 检测安装是否完成
$ etcdctl version
etcdctl version: 3.5.11
API version: 3.5
# 查看帮助信息
$ etcdctl -h 
```

# 使用

## 增删改查

**这里假设已经部署了一个集群，且端口号分别为23791，23792，23793**

```bash
# 查询集群信息
$ etcdctl --endpoints=127.0.0.1:23791,127.0.0.1:23792,127.0.0.1:23793 member list
...

# 这里向其中一个节点添加键值对 put
# put 也可以对现有的key进行更新
# etcd有mvvc机制 
$ etcdctl --endpoints=127.0.0.1:23791 put name kdy
OK
# 这里再去查另一个节点的数据 get
# 可以看到已经同步到了另一个节点
$ etcdctl --endpoints=127.0.0.1:23792 get name
name
kdy 
# 请求第三个节点以此来删除一个键值对 del
# 可以使用 --prev-kv来实现pop效果(删除同时返回一个值)
$ etcdctl --endpoints=127.0.0.1:23793 del name
1

# 这里添加三个key
$ etcdctl --endpoints=127.0.0.1:23793 put /name1 n1
OK
$ etcdctl --endpoints=127.0.0.1:23793 put /name2 n2
OK
$ etcdctl --endpoints=127.0.0.1:23793 put /name3 n3
OK
# 再去另一个节点来验证是否同步
# /name1 /name3 表示一个区间 左闭右开
$ etcdctl --endpoints=127.0.0.1:23792 get /name1 /name3
/name1
n1
/name2
n2
# prefix表示获取以什么开头的键
$ etcdctl --endpoints=127.0.0.1:23792 get --prefix /name
/name1
n1
/name2
n2
/name3
n3
# 使用 --limit 来限制返回的个数
$ etcdctl --endpoints=127.0.0.1:23792 get --prefix /name --limit=2
/name1
n1
/name2
n2
# 使用 -w=json 来输出json格式的数据
$ etcdctl --endpoints=127.0.0.1:23792 get --prefix /name --limit=3 -w=json | jq .
{
  "header": {
    "cluster_id": 10316109323310760000, # 集群id
    "member_id": 15168875803774600000, # 请求的etcd节点id
     # etcd服务端的当前全局数据版本号 
     # 对任一一个key的put或者delete操作都会使revision自增 1
     # 1 是etcd的保留版本号 所以用户的key版本号从2开始 !!!
    "revision": 8,
     # etcd etcd当前raft主节点任期号
    "raft_term": 8
  },
  "kvs": [
    {
      "key": "L25hbWUx",
      # 这个key创建的时候全局版本号
      "create_revision": 6,
      # 当前key最后一次修改时全局版本号
      # 可以使用 --rev=xxx 来实现访问对应版本的key
      "mod_revision": 6,
      # 当前key的版本号 对keyput会使该字段自增1
      # key删除后再创建又会从1开始计数
      "version": 1,
      "value": "bjE="
    },
    {
      "key": "L25hbWUy",
      "create_revision": 7,
      "mod_revision": 7,
      "version": 1,
      "value": "bjI="
    },
    {
      "key": "L25hbWUz",
      "create_revision": 8,
      "mod_revision": 8,
      "version": 1,
      "value": "bjM="
    }
  ],
  "count": 3
}
```

## watch 和 lease

watch来监测一个值是否改变改变就会输出最新的值

**zookeeper 的watch是一次性的 而etcd是持久的**

```bash
$ etcdctl --endpoints=127.0.0.1:23792 put /watch1 1
OK
# 这里去另一个集群watch来验证集群的同步
$ etcdctl --endpoints=127.0.0.1:23791 watch /watch1
# 这个时候我们再开一个终端来更新这个值
$ etcdctl --endpoints=127.0.0.1:23792 put /watch1 11
# 回到原终端可以看到输出了更新
PUT
/watch1
11
```

lease(租约) 类似于redis的expire 主要是赋予一个键的过期时间

附:**租约原理**:https://segmentfault.com/a/1190000021787065

```bash
# 这里赋予一个租约 30s
$ etcdctl --endpoints=127.0.0.1:23792 lease grant 30
lease 41ce8c579a16e015 granted with TTL(30s)
# 查看租约信息
$ etcdctl --endpoints=127.0.0.1:23792 lease timetolive 41ce8c579a16e015
lease 41ce8c579a16e015 granted with TTL(200s), remaining(183s)
# 绑定到键
$ etcdctl --endpoints=127.0.0.1:23792 put --lease=41ce8c579a16e015 /watch1 1
OK
# 续约 打开续约后终端会阻塞
$ etcdctl --endpoints=127.0.0.1:23792 lease keep-alive 41ce8c579a16e015
lease 41ce8c579a16e015 keepalived with TTL(200)
lease 41ce8c579a16e015 keepalived with TTL(200)
lease 41ce8c579a16e015 keepalived with TTL(200)
lease 41ce8c579a16e015 keepalived with TTL(200)
^C
# 撤销 同时删除所有绑定的key
$ etcdctl --endpoints=127.0.0.1:23792 lease revoke 41ce8c579a16e015
lease 41ce8c579a16e015 revoked
```

## 权限管理

etcd是有完整的权限系统的 在做相关权限操作之前先检查auth是否开启

```bash
$ etcdctl --endpoints=127.0.0.1:23791 auth status
Authentication Status: false
AuthRevision: 1
# 可以看到是false
# 可以使用 auth enable开启权限
# 不过在开启权限之前我们得先有一个用户
# 这里拿root来举例 (这里我密码设置为123456)
$ etcdctl --endpoints=127.0.0.1:23791 user add root
Password of root:
Type password of root again for confirmation:
User root created
# 创建用户过后还要给这个用户一个角色
$
# 配置过后就可以打开auth了

```
