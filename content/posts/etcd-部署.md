---
title: "Etcd 部署"
date: 2023-12-10T22:42:37+08:00
draft: true
tags:
 - Hello,world
 - etcd
---
# 开始

## **etcd 介绍**

2013 年 6 月，CoreOS 发起了 etcd 项目。etcd 使用 Go 语言实现，是分布式系统中重要的基础组件，目前最新版本为 V3.4.9。etcd 可以用来构建高可用的分布式键值数据库。

截自githubt etcd仓库(https://github.com/etcd-io/etcd) 由此可见etcd的重要性

![](https://raw.githubusercontent.com/woshikedayaa/blog-image-bed/main/image-20231208192020620.png)

**etcd 应用场景**

etcd 在**稳定性、可靠性和可伸缩性**表现极佳，同时也为云原生应用系统提供了协调机制。etcd 经常用于**服务注册与发现**的场景，此外还有键值对存储、消息发布与订阅、分布式锁等场景。利用 Etcd 的特性，应用程序可以在集群中共享信息、配置或作服务发现，Etcd 会在集群的各个节点中复制这些数据并保证这些数据始终正确。

# 通过compose搭建集群

**这里不用单体模式的一个原因就是集群可能用得更多 而且后面学习可以用**

````yaml
# ref : https://blog.csdn.net/qq_30145355/article/details/115468341
# 静态配置
version: "3.0"

networks:
  etcd-net:           # 网络
    driver: bridge    # 桥接模式

volumes:
  etcd1_data:         # 挂载到本地的数据卷名
    driver: local
  etcd2_data:
    driver: local
  etcd3_data:
    driver: local
###
### etcd 其他环境配置见：https://doczhcn.gitbook.io/etcd/index/index-1/configuration
###
services:
  etcd1:
    image: bitnami/etcd:latest  # 镜像
    container_name: etcd1       # 容器名 --name
    restart: always             # 总是重启
    networks:
      - etcd-net                # 使用的网络 --network
    ports:                      # 端口映射 -p
      - "23791:2379"
      - "23801:2380"
    environment:                # 环境变量 --env
      - ALLOW_NONE_AUTHENTICATION=yes                       # 允许不用密码登录
      - ETCD_NAME=etcd1                                     # etcd 的名字
      - ETCD_INITIAL_ADVERTISE_PEER_URLS=http://etcd1:2380  # 列出这个成员的伙伴 URL 以便通告给集群的其他成员
      - ETCD_LISTEN_PEER_URLS=http://0.0.0.0:2380           # 用于监听伙伴通讯的URL列表
      - ETCD_LISTEN_CLIENT_URLS=http://0.0.0.0:2379         # 用于监听客户端通讯的URL列表
      - ETCD_ADVERTISE_CLIENT_URLS=http://etcd1:2379        # 列出这个成员的客户端URL，通告给集群中的其他成员
      - ETCD_INITIAL_CLUSTER_TOKEN=etcd-cluster             # 在启动期间用于 etcd 集群的初始化集群记号
      - ETCD_INITIAL_CLUSTER=etcd1=http://etcd1:2380,etcd2=http://etcd2:2380,etcd3=http://etcd3:2380 # 为启动初始化集群配置
      - ETCD_INITIAL_CLUSTER_STATE=new                      # 初始化集群状态
    volumes:
      - etcd1_data:/bitnami/etcd                            # 挂载的数据卷

  etcd2:
    image: bitnami/etcd:latest
    container_name: etcd2
    restart: always
    networks:
      - etcd-net
    ports:
      - "23792:2379"
      - "23802:2380"
    environment:
      - ALLOW_NONE_AUTHENTICATION=yes
      - ETCD_NAME=etcd2
      - ETCD_INITIAL_ADVERTISE_PEER_URLS=http://etcd2:2380
      - ETCD_LISTEN_PEER_URLS=http://0.0.0.0:2380
      - ETCD_LISTEN_CLIENT_URLS=http://0.0.0.0:2379
      - ETCD_ADVERTISE_CLIENT_URLS=http://etcd2:2379
      - ETCD_INITIAL_CLUSTER_TOKEN=etcd-cluster
      - ETCD_INITIAL_CLUSTER=etcd1=http://etcd1:2380,etcd2=http://etcd2:2380,etcd3=http://etcd3:2380
      - ETCD_INITIAL_CLUSTER_STATE=new
    volumes:
      - etcd2_data:/bitnami/etcd

  etcd3:
    image: bitnami/etcd:latest
    container_name: etcd3
    restart: always
    networks:
      - etcd-net
    ports:
      - "23793:2379"
      - "23803:2380"
    environment:
      - ALLOW_NONE_AUTHENTICATION=yes
      - ETCD_NAME=etcd3
      - ETCD_INITIAL_ADVERTISE_PEER_URLS=http://etcd3:2380
      - ETCD_LISTEN_PEER_URLS=http://0.0.0.0:2380
      - ETCD_LISTEN_CLIENT_URLS=http://0.0.0.0:2379
      - ETCD_ADVERTISE_CLIENT_URLS=http://etcd3:2379
      - ETCD_INITIAL_CLUSTER_TOKEN=etcd-cluster
      - ETCD_INITIAL_CLUSTER=etcd1=http://etcd1:2380,etcd2=http://etcd2:2380,etcd3=http://etcd3:2380
      - ETCD_INITIAL_CLUSTER_STATE=new
    volumes:
      - etcd3_data:/bitnami/etcd
````

![](https://raw.githubusercontent.com/woshikedayaa/blog-image-bed/main/image-20231210195726077.png)

上面搭建的是静态模式 其实etcd有三种模式

* 静态模式(线下可以这么干)

  ```text
  必须要知道你启动的多少节点，同时节点的地址也必须知道。
  不常用 无法实现动态扩展
  
  👇访问任意一个节点来获取集群的信息
  $ etcdctl --endpoints=127.0.0.1:23791 member list
  ade526d28b1f92f7, started, etcd1, http://etcd1:2380, http://etcd1:2379,false
  bd388e7810915853, started, etcd3, http://etcd3:2380, http://etcd3:2379,false
  d282ac2ce600c1ce, started, etcd2, http://etcd2:2380, http://etcd2:2379,false
  ```
  
* 基于etcd的动态模式(常用)

* dns模式

  ## etcd集群发现模式

  ### 实现原理

  Discovery service protocol帮助新的etcd成员使用共享URL在集群引导阶段发现所有其他成员。
  该协议使用新的发现令牌来引导一个唯一的etcd集群。一个发现令牌只能代表一个etcd集群。只要此令牌上的发现协议启动，即使它中途失败，也不能用于引导另一个etcd集群
  
  **基于etcd服务发现模式有两个方案**
  
  ### 私有etcd
  
  **相关文档:https://www.zhaowenyu.com/etcd-doc/ops/etcd-discovery-etcd.html**
  
   一个是先搭建一个私有的etcd 然后将改etcd作为用于发现服务的etcd
  
  ```bash
  $ docker network create etcd-net
  # 假如我先搭建了一个etcd 并且客户端端口为 23790
  $ docker run --name=etcd-discovery -itd \
  -v etcd0_data:/bitnami/etcd \
  -e ALLOW_NONE_AUTHENTICATION=yes \
  -e ETCD_NAME=etcd-discovery \
  -e ETCD_INITIAL_ADVERTISE_PEER_URLS=http://etcd-discovery:2380 \
  -e ETCD_LISTEN_PEER_URLS=http://0.0.0.0:2380 \
  -e ETCD_LISTEN_CLIENT_URLS=http://0.0.0.0:2379 \
  -e ETCD_ADVERTISE_CLIENT_URLS=http://etcd-discovery:2379 \
  -e ETCD_INITIAL_CLUSTER_TOKEN=etcd-cluster \
  -e ETCD_INITIAL_CLUSTER_STATE=new \
  --network=etcd-net \
  -p 23790:2379 \
  -p 23800:2380 \
  bitnami/etcd
  # 这里搭建的etcd为docker搭建 并且名字为etcd-discovery(下文会用)
  # 这里引入一个 uuid生成工具 后面会用到
  $ sudo apt install uuid 
  $ uuid
  0b0bdd00-975c-11ee-9929-00155d636dd2 # 记住这个uuid(下文会用)
  # 这个value填你要注册的集群数量 这里我compose 写的是3个我就填3 
  # 多余的会采用proxy的模式来实现
  $ curl -X \
  PUT \
  http://127.0.0.1:23790/v2/keys/discovery/0b0bdd00-975c-11ee-9929-00155d636dd2/_config/size -d value=3 
  ```
  
  经过上面操作后就已经在一个私有的etcd注册了一个集群信息
  
  接下来写一个docker-compose.yml来启动其他集群
  
  ```yaml
  # ref : https://blog.csdn.net/qq_30145355/article/details/115468341
  # etcd动态发现配置
  version: "3.0"
  
  networks:
    etcd-net:           # 网络
      driver: bridge    # 桥接模式
  
  volumes:
    etcd1_data:         # 挂载到本地的数据卷名
      driver: local
    etcd2_data:
      driver: local
    etcd3_data:
      driver: local
  services:
    etcd1:
      image: bitnami/etcd:latest  # 镜像
      container_name: etcd1       # 容器名 --name
      restart: always             # 总是重启
      networks:
        - etcd-net                # 使用的网络 --network
      ports:                      # 端口映射 -p
        - "23791:2379"
        - "23801:2380"
      environment:                # 环境变量 --env
        - ALLOW_NONE_AUTHENTICATION=yes                       # 允许不用密码登录
        - ETCD_NAME=etcd1                                     # etcd 的名字
        - ETCD_INITIAL_ADVERTISE_PEER_URLS=http://etcd1:2380  # 列出这个成员的伙伴 URL 以便通告给集群的其他成员
        - ETCD_LISTEN_PEER_URLS=http://0.0.0.0:2380           # 用于监听伙伴通讯的URL列表
        - ETCD_LISTEN_CLIENT_URLS=http://0.0.0.0:2379         # 用于监听客户端通讯的URL列表
        - ETCD_ADVERTISE_CLIENT_URLS=http://etcd1:2379        # 列出这个成员的客户端URL，通告给集群中的其他成员
        - ETCD_INITIAL_CLUSTER_TOKEN=etcd-cluster             # 在启动期间用于 etcd 集群的初始化集群记号
        #- ETCD_INITIAL_CLUSTER=etcd1=http://etcd1:2380,etcd2=http://etcd2:2380,etcd3=http://etcd3:2380 # 为启动初始化集群配置
        - ETCD_DISCOVERY=http://etcd-discovery:2379/v2/keys/discovery/0b0bdd00-975c-11ee-9929-00155d636dd2
        - ETCD_INITIAL_CLUSTER_STATE=new                      # 初始化集群状态
      volumes:
        - etcd1_data:/bitnami/etcd                            # 挂载的数据卷
  
    etcd2:
      image: bitnami/etcd:latest
      container_name: etcd2
      restart: always
      networks:
        - etcd-net
      ports:
        - "23792:2379"
        - "23802:2380"
      environment:
        - ALLOW_NONE_AUTHENTICATION=yes
        - ETCD_NAME=etcd2
        - ETCD_INITIAL_ADVERTISE_PEER_URLS=http://etcd2:2380
        - ETCD_LISTEN_PEER_URLS=http://0.0.0.0:2380
        - ETCD_LISTEN_CLIENT_URLS=http://0.0.0.0:2379
        - ETCD_ADVERTISE_CLIENT_URLS=http://etcd2:2379
        - ETCD_INITIAL_CLUSTER_TOKEN=etcd-cluster
        #- ETCD_INITIAL_CLUSTER=etcd1=http://etcd1:2380,etcd2=http://etcd2:2380,etcd3=http://etcd3:2380
        - ETCD_DISCOVERY=http://etcd-discovery:2379/v2/keys/discovery/0b0bdd00-975c-11ee-9929-00155d636dd2
        - ETCD_INITIAL_CLUSTER_STATE=new
      volumes:
        - etcd2_data:/bitnami/etcd
  
    etcd3:
      image: bitnami/etcd:latest
      container_name: etcd3
      restart: always
      networks:
        - etcd-net
      ports:
        - "23793:2379"
        - "23803:2380"
      environment:
        - ALLOW_NONE_AUTHENTICATION=yes
        - ETCD_NAME=etcd3
        - ETCD_INITIAL_ADVERTISE_PEER_URLS=http://etcd3:2380
        - ETCD_LISTEN_PEER_URLS=http://0.0.0.0:2380
        - ETCD_LISTEN_CLIENT_URLS=http://0.0.0.0:2379
        - ETCD_ADVERTISE_CLIENT_URLS=http://etcd3:2379
        - ETCD_INITIAL_CLUSTER_TOKEN=etcd-cluster
        #- ETCD_INITIAL_CLUSTER=etcd1=http://etcd1:2380,etcd2=http://etcd2:2380,etcd3=http://etcd3:2380
        - ETCD_DISCOVERY=http://etcd-discovery:2379/v2/keys/discovery/0b0bdd00-975c-11ee-9929-00155d636dd2
        - ETCD_INITIAL_CLUSTER_STATE=new
      volumes:
        - etcd3_data:/bitnami/etcd
  ```
  
  注意其中的更改 启动即可启动私有etcd环境下的内容
  
  ### 公共etcd
  
  公共的 discovery 就是通过 CoreOS 提供的公共 discovery 服务申请 token。
  
  ```bash
  $ curl https://discovery.etcd.io/new?size=3
  https://discovery.etcd.io/{uuid}
  # 以上命令会生成一个链接样式的 token，参数 size 代表要创建的集群大小，即: 有多少集群节点。
  ```
  
  然后再基于这个链接来配置compose中的配置
  
  ```yaml
  # ref : https://blog.csdn.net/qq_30145355/article/details/115468341
  # etcd动态发现配置
  version: "3.0"
  
  networks:
    etcd-net:           # 网络
      driver: bridge    # 桥接模式
  
  volumes:
    etcd1_data:         # 挂载到本地的数据卷名
      driver: local
    etcd2_data:
      driver: local
    etcd3_data:
      driver: local
  services:
    etcd1:
      image: bitnami/etcd:latest  # 镜像
      container_name: etcd1       # 容器名 --name
      restart: always             # 总是重启
      networks:
        - etcd-net                # 使用的网络 --network
      ports:                      # 端口映射 -p
        - "23791:2379"
        - "23801:2380"
      environment:                # 环境变量 --env
        - ALLOW_NONE_AUTHENTICATION=yes                       # 允许不用密码登录
        - ETCD_NAME=etcd1                                     # etcd 的名字
        - ETCD_INITIAL_ADVERTISE_PEER_URLS=http://etcd1:2380  # 列出这个成员的伙伴 URL 以便通告给集群的其他成员
        - ETCD_LISTEN_PEER_URLS=http://0.0.0.0:2380           # 用于监听伙伴通讯的URL列表
        - ETCD_LISTEN_CLIENT_URLS=http://0.0.0.0:2379         # 用于监听客户端通讯的URL列表
        - ETCD_ADVERTISE_CLIENT_URLS=http://etcd1:2379        # 列出这个成员的客户端URL，通告给集群中的其他成员
        - ETCD_INITIAL_CLUSTER_TOKEN=etcd-cluster             # 在启动期间用于 etcd 集群的初始化集群记号
        #- ETCD_INITIAL_CLUSTER=etcd1=http://etcd1:2380,etcd2=http://etcd2:2380,etcd3=http://etcd3:2380 # 为启动初始化集群配置
        - ETCD_DISCOVERY=https://discovery.etcd.io/{uuid}
        - ETCD_INITIAL_CLUSTER_STATE=new                      # 初始化集群状态
      volumes:
        - etcd1_data:/bitnami/etcd                            # 挂载的数据卷
  
    etcd2:
      image: bitnami/etcd:latest
      container_name: etcd2
      restart: always
      networks:
        - etcd-net
      ports:
        - "23792:2379"
        - "23802:2380"
      environment:
        - ALLOW_NONE_AUTHENTICATION=yes
        - ETCD_NAME=etcd2
        - ETCD_INITIAL_ADVERTISE_PEER_URLS=http://etcd2:2380
        - ETCD_LISTEN_PEER_URLS=http://0.0.0.0:2380
        - ETCD_LISTEN_CLIENT_URLS=http://0.0.0.0:2379
        - ETCD_ADVERTISE_CLIENT_URLS=http://etcd2:2379
        - ETCD_INITIAL_CLUSTER_TOKEN=etcd-cluster
        #- ETCD_INITIAL_CLUSTER=etcd1=http://etcd1:2380,etcd2=http://etcd2:2380,etcd3=http://etcd3:2380
        - ETCD_DISCOVERY=https://discovery.etcd.io/{uuid}
        - ETCD_INITIAL_CLUSTER_STATE=new
      volumes:
        - etcd2_data:/bitnami/etcd
  
    etcd3:
      image: bitnami/etcd:latest
      container_name: etcd3
      restart: always
      networks:
        - etcd-net
      ports:
        - "23793:2379"
        - "23803:2380"
      environment:
        - ALLOW_NONE_AUTHENTICATION=yes
        - ETCD_NAME=etcd3
        - ETCD_INITIAL_ADVERTISE_PEER_URLS=http://etcd3:2380
        - ETCD_LISTEN_PEER_URLS=http://0.0.0.0:2380
        - ETCD_LISTEN_CLIENT_URLS=http://0.0.0.0:2379
        - ETCD_ADVERTISE_CLIENT_URLS=http://etcd3:2379
        - ETCD_INITIAL_CLUSTER_TOKEN=etcd-cluster
        #- ETCD_INITIAL_CLUSTER=etcd1=http://etcd1:2380,etcd2=http://etcd2:2380,etcd3=http://etcd3:2380
        - ETCD_DISCOVERY=https://discovery.etcd.io/{uuid}
        - ETCD_INITIAL_CLUSTER_STATE=new
      volumes:
        - etcd3_data:/bitnami/etcd
  ```
  
  最后再启动compose即可实现etcd基于动态发现的集群
  
  ## DNS模式
  
  略 详见：https://www.zhaowenyu.com/etcd-doc/ops/etcd-discovery-dns.html

​	


