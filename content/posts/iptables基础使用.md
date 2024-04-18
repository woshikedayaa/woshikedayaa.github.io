---
title: Iptables基础使用
date: 2024-04-17T21:04:03+08:00
draft: false
tags:
  - linux
  - tool
---

# 开始
iptables 是 Linux 下可以在用户空间管理内核网络的一个工具

这里 摘选一下 [Wiki](https://zh.wikipedia.org/wiki/Iptables)的一句话
```text
iptables是运行在用户空间的应用软件，通过控制Linux内核netfilter模块，来管理网络数据包的处理和转发。
```
linux的包过滤功能，即linux防火墙，它由netfilter 和 iptables 两个组件组成。

netfilter 组件也称为内核空间，是内核的一部分，由一些信息包过滤表组成，这些表包含内核用来控制信息包过滤处理的规则集。

iptables 组件是一种工具，也称为用户空间，它使插入、修改和除去信息包过滤表中的规则变得容易。

基于此 我们就可以用 它来管理 Linux 内核网络

# 安装
在现在通常Linux 发行版中 例如 Debian12 已经自带了iptables 

如果没有安装 请自行查询相关发行版文档安装 iptables

## 验证安装
```bash
$ iptables -V
iptables v1.8.7 (nf_tables)
# 这里只是示例 有可能会随着版本更新所变化
```

# 基础知识

## 四表五链
- 四表 : raw, mangle, filter, nat
- 五链(指mangle表的五链): PREROUTING, INPUT, OUTPUT, FORWARD, POSTROUTING
>[!CAUTION]
>这里不止mangle表有链，其他表也有链，只不过mangle表的链是最多的
>这里放下一张图片帮助理解 (非作者原创)

![iptables详细](https://wooyun.js.org/images_result/images/2014071403584011695.jpg)

## raw 表
这个表非常少用 主要是处理异常 这里不介绍

## mangle 表
这个是 网络通过 raw 表后到达的第一个表 进入这个表的第一个链是 PREROUTING 
该表允许我们更改IP标头。例如，我们可以更改输入数据包中的TTL值
### PREROUTING
路由前链，在处理路由规则前通过此链，通常用于目的地址转换（DNAT）
这个链内部判断这个网络请求的地址是不是请求的是本地网卡地址或者是不是本机发出的
如果是就进入 *INPUT* 链条 如果不是就进入 FORWARD 链
>[!TIP]
>这里FORWARD链是否转发这个请求取决于系统设置
>```bash
>$ sysctl -a | grep "net.ipv4.ip_forward"
>net.ipv4.ip_forward = 1 # 如果显示这个代表是开启的
>...
>```
>如果没有开启 可以在 /etc/sysctl.conf 文件添加一行 `net.ipv4.ip_forward=1`
>运行下列命令来应用
>```bash
>$ sysctl -p
>```

### INPUT
这里就是如果这个请求如果是本机发出或者来自外部的网络请求到本机的请求就进入这个链

### POSTROUTING
这个链条是所有的数据包都会经过 不论是走的 FORWARD 链还是走的INPUT 链条 经过处理后的数据包 最终都会走到这个链条来处理

### OUTPUT
这个链条是进入INPUT 链条处理完毕过后再走到的链条 这个时候数据包刚从raw表的OUTPUT链处理出来

### FORWARD
这个链条通常是针对非本机的请求的转发出去
转发出去过后的包会经过 filter 表的FORWARD 链，mangle的POSTROUTING 链

## filter 表
这个表是用得最多的一张表 通常用于过滤数据包
这里就不多介绍了
主要有三个链：
-  INPUT，输入链。发往本机的数据包通过此链。
-  OUTPUT，输出链。从本机发出的数据包通过此链。
-  FORWARD，转发链。本机转发的数据包通过此链。

## nat 表
nat表如其名，用于[地址转换](https://zh.wikipedia.org/wiki/%E7%BD%91%E7%BB%9C%E5%9C%B0%E5%9D%80%E8%BD%AC%E6%8D%A2 "网络地址转换")操作。
mangle表侧重每一个数据包，nat表侧重连接。
主要有三个链：
- PREROUTING，路由前链，在处理路由规则前通过此链，通常用于目的地址转换（DNAT）。
- POSTROUTING，路由后链，完成路由规则后通过此链，通常用于源地址转换（SNAT）。
- OUTPUT，输出链，类似PREROUTING，但是处理本机发出的数据包。

这里放上一张图来帮助理解 （非原创）
![iptables工作流程](https://pic2.zhimg.com/80/v2-63126807f733612004f8e666837bb13d_1440w.webp)

# 命令使用
```bash
$ iptables [ -t 表名] 命令选项 [链名] [条件匹配] [-j 目标动作或跳转]
```

>[!TIP]
>表名  -t 是可选的 如果不指定表名 默认是 filter  表

## 先看一下help
```bash
$ sudo iptables --help
iptables v1.8.7

Usage: iptables -[ACD] chain rule-specification [options]
	iptables -I chain [rulenum] rule-specification [options]
	iptables -R chain rulenum rule-specification [options]
	iptables -D chain rulenum [options]
	iptables -[LS] [chain [rulenum]] [options]
	iptables -[FZ] [chain] [options]
	iptables -[NX] chain
	iptables -E old-chain-name new-chain-name
	iptables -P chain target [options]
	iptables -h (print this help information)

Commands:
Either long or short options are allowed.
  --append  -A chain		Append to chain
  --check   -C chain		Check for the existence of a rule
  --delete  -D chain		Delete matching rule from chain
  --delete  -D chain rulenum
				Delete rule rulenum (1 = first) from chain
  --insert  -I chain [rulenum]
				Insert in chain as rulenum (default 1=first)
  --replace -R chain rulenum
				Replace rule rulenum (1 = first) in chain
  --list    -L [chain [rulenum]]
				List the rules in a chain or all chains
  --list-rules -S [chain [rulenum]]
				Print the rules in a chain or all chains
  --flush   -F [chain]		Delete all rules in  chain or all chains
  --zero    -Z [chain [rulenum]]
				Zero counters in chain or all chains
  --new     -N chain		Create a new user-defined chain
  --delete-chain
	     -X [chain]		Delete a user-defined chain
  --policy  -P chain target
				Change policy on chain to target
  --rename-chain
	     -E old-chain new-chain
				Change chain name, (moving any references)
Options:
    --ipv4	-4		Nothing (line is ignored by ip6tables-restore)
    --ipv6	-6		Error (line is ignored by iptables-restore)
[!] --proto	-p proto	protocol: by number or name, eg. `tcp'
[!] --source	-s address[/mask][...]
				source specification
[!] --destination -d address[/mask][...]
				destination specification
[!] --in-interface -i input name[+]
				network interface name ([+] for wildcard)
 --jump	-j target
				target for rule (may load target extension)
  --goto      -g chain
			       jump to chain with no return
  --match	-m match
				extended match (may load extension)
  --numeric	-n		numeric output of addresses and ports
[!] --out-interface -o output name[+]
				network interface name ([+] for wildcard)
  --table	-t table	table to manipulate (default: `filter')
  --verbose	-v		verbose mode
  --wait	-w [seconds]	maximum wait to acquire xtables lock before give up
  --wait-interval -W [usecs]	wait time to try to acquire xtables lock
				default is 1 second
  --line-numbers		print line numbers when listing
  --exact	-x		expand numbers (display exact values)
[!] --fragment	-f		match second or further fragments only
  --modprobe=<command>		try to insert modules using this command
  --set-counters PKTS BYTES	set the counter during insert/append
[!] --version	-V		print package version.
```
## 查
```bash
# -L 列出filter表的所有规则(rule)
$ iptables -L 

# -L <chain> 列出filter表的INPUT 链的所有规则
$ iptables -L INPUT

# -t 列出mangle表的所有规则
$ iptables -t mangle -L

# -v -vv --verbose 列出mangle表的所有规则(详细)
$ iptables -t mangle -v -L # 如果需要更加详细 可以使用 -vv

# -n --numeric 列出nat表的所有规则 用数字显示输出结果
# 显示主机的 IP地址而不是主机名
$ iptables -t nat -L -n

# --line-number 查看规则列表的同时 显示规则在链中的顺序号 --list-number
$ iptables -t nat -L -n --list-number 

```

## 增
```bash
# -N 在filter表新增一条名为CUSTOM的规则链 可以通过 iptables -L 列出
$ iptables -N CUSTOM

# -A 在filter表的CUSTOM链追加一条规则
$ iptables -A CUSTOM -j DROP # 关于DROP规则 后文再讲

# -I 在filter表的CUSTOM链在下表为0的位置插入一条规则
# 如果要在头部插入 1 是可选可不选的 
# iptables 的rule下标是从 1 开始的 ， 不是从0开始的
# 如果是插入的话就
$ iptables -I CUSTOM 1
```