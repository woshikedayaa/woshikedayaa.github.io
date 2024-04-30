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
$ iptables --help
iptables v1.8.7
# -ACD 见下文
Usage: iptables -[ACD] chain rule-specification [options]
	# 在指定链中插入（--insert）一条新的规则，默认在链的开头插入
	iptables -I chain [rulenum] rule-specification [options]
	# 修改、替换（--replace）指定链中的一条规则，按规则序号或内容确定
	iptables -R chain rulenum rule-specification [options]
	# 删除（--delete）指定链中的某一条规则，按规则序号或内容确定要删除的规则
	iptables -D chain rulenum [options]
	# -L 列出规则链的所有规则
	# -S 以命令的方式列出
	# 两者具体差别可以自行试验
	iptables -[LS] [chain [rulenum]] [options]
	# -F -Z 都是清除一个链条的所有规则 
	# 如果没有指定链 则清除这个表的所有规则
	iptables -[FZ] [chain] [options]
	# -N 新建一个链
	# -X 删除一个链条(规则也会被清除)
	iptables -[NX] chain
	# 修改（--rename-chain）一个链条的名称 从 old-chain-name 改变到 new-chain-name
	# 之前指向这个链的规则名字也会跟着变
	iptables -E old-chain-name new-chain-name
	# 指定一个链条的默认规则(--policy)
	iptables -P chain target [options]
	# ...................................
	iptables -h (print this help information)

Commands: # 这里我只列出一些没讲到的或者需要细分的
Either long or short options are allowed.
  # 追加一个规则到一条链里面
  --append  -A chain		Append to chain
  # 查看一个规则是否存在
  --check   -C chain		Check for the existence of a rule
  # 删除一个规则
  # 第一个方式是指定这个规则的详细
  # 第二个方式是规则的下标
  --delete  -D chain		Delete matching rule from chain
  --delete  -D chain rulenum
				Delete rule rulenum (1 = first) from chain
  # 替换对应rulenum为一个新的规则
  # 不咋用 可以用 -D + -I 来实现
  --replace -R chain rulenum
				Replace rule rulenum (1 = first) in chain

	# 这里都是新建(修改)规则的时候会用到
	Options: # 这里我只列出后面不会讲到的 且有用的
	# 这两个不用说
    --ipv4	-4		Nothing (line is ignored by ip6tables-restore)
    --ipv6	-6		Error (line is ignored by iptables-restore)
    # 指定协议
    --proto	-p proto	protocol: by number or name, eg. `tcp`
    # 请求来源 （不填是 anyway ）
    --source	-s address[/mask][...]
				source specification
	# 目标地址
    --destination -d address[/mask][...]
				destination specification
	# 这个是说明输入的网口 如果输入不是这个网口就不匹配
    --in-interface -i input name[+]
				network interface name ([+] for wildcard)
	# 选择这个规则如果匹配 选择的 规则链/控制方式
	# 内置的有
	# ACCEPT：允许数据包通过。 
	# DROP：直接丢弃数据包，不给出任何回应信息。 
	# REJECT：拒绝数据包通过，必须时会给数据发送端一个响应信息。 
	# LOG：在/var/log/messages 文件中记录日志信息，然后将数据包传递给下一条规则。 
	# QUEUE：防火墙将数据包移交到用户空间 
	# RETURN：防火墙停止执行当前链中的后续Rules，并返回到调用链(the calling chain) 
    --jump	-j target
				target for rule (may load target extension)
    # 后面详细讲
    --match	-m match
				extended match (may load extension)
	# 输出地址以数字化的输出 例如可以把 localhost 转化成 127.0.0.1
    --numeric	-n		numeric output of addresses and ports
    # 这个是说明输出的网口 如果输出不是这个网口就不匹配
    --out-interface -o output name[+]
				network interface name ([+] for wildcard)
	# 指定表
    --table	-t table	table to manipulate (default: `filter`)

	# 这两个是查的时候用的
	# 列出详细信息
	--verbose	-v		verbose mode
	# 注：-vv 可以更详细
	
	# 列出规则的时候同时列出规则的rulenum
	--line-numbers		print line numbers when listing
```

## -m 详解
条件匹配分为基本匹配和扩展匹配，拓展匹配又分为隐式扩展和显示扩展。

基本匹配包括：
```sh
-p    指定规则协议，如tcp, udp,icmp等，可以使用all来指定所有协议
-s    指定数据包的源地址参数，可以使IP地址、网络地址、主机名
-d    指定目的地址
-i    输入接口
-o    输出接口
```
隐式扩展包括：(图片非原创)
![](https://wooyun.js.org/images_result/images/2014071403584061278.jpg)
常用显式扩展:(图片非原创)
![](https://wooyun.js.org/images_result/images/2014071403582276903.jpg)

## 简单使用
删除iptables现有规则(filter 表)
```sh
$ sudo iptables –F 
```

查看iptables规则(filter 表)
```sh
$ sudo iptables -L
```

增加一条规则到最后
```sh
$ sudo iptables -t filter -A INPUT -i eth0 -p tcp --dport 80 -m state --state NEW,ESTABLISHED -j ACCEPT

# 注 这里其实是有层次关系的
# - 先指定表 -t filter
# - 再指定规则链条 -A(-I) INPUT
# - 再写规则 -i eth0 -p tcp --dport 80 -m state --state NEW,ESTABLISHED
# - 最后指定如果匹配到了这一条规则 选择的 规则链/控制方式 -j ACCPET
```

添加一条规则到指定位置
```sh
$ sudo iptables -I INPUT 2 -i eth0 -p tcp --dport 80 -m state --state NEW,ESTABLISHED -j ACCEPT 
```

删除一条规则
```sh
$ sudo iptabels -D INPUT 2 
```

修改一条规则
```sh
$ sudo iptables -R INPUT 3 -i eth0 -p tcp --dport 80 -m state --state NEW,ESTABLISHED -j ACCEPT 
```

设置默认策略
```sh
$ sudo iptables -P INPUT DROP 
```

# 实际应用
允许远程主机进行SSH连接
```sh
$ sudo iptables -A INPUT -i eth0 -p tcp --dport 22 -m state --state NEW,ESTABLISHED -j ACCEPT 
$ sudo iptables -A OUTPUT -o eth0 -p tcp --sport 22 -m state --state ESTABLISHED -j ACCEPT 
```

允许本地主机进行SSH连接
```sh
$ sudo iptables -A OUTPUT -o eth0 -p tcp --dport 22 -m state --state NEW,ESTABLISHED -j ACCEPT 
$ sudo iptables -A INTPUT -i eth0 -p tcp --sport 22 -m state --state ESTABLISHED -j ACCEPT 
```

允许HTTP请求
```sh
$ sudo iptables -A INPUT -i eth0 -p tcp --dport 80 -m state --state NEW,ESTABLISHED -j ACCEPT 
$ sudo iptables -A OUTPUT -o eth0 -p tcp --sport 80 -m state --state ESTABLISHED -j ACCEPT 
```

限制ping 192.168.146.3主机的数据包数，平均2/s个，最多不能超过3个
```sh
$ sudo iptables -A INPUT -i eth0 -d 192.168.146.3 -p icmp --icmp-type 8 -m limit --limit 2/second --limit-burst 3 -j ACCEPT 
```

限制SSH连接速率(默认策略是DROP)
```sh
$ sudo iptables -I INPUT 1 -p tcp --dport 22 -d 192.168.146.3 -m state --state ESTABLISHED -j ACCEPT  
$ sudo iptables -I INPUT 2 -p tcp --dport 22 -d 192.168.146.3 -m limit --limit 2/minute --limit-burst 2 -m state --state NEW -j ACCEPT 
```

防止syn攻击
```sh
# 思路1：限制syn的请求速度（这个方式需要调节一个合理的速度值，不然会影响正常用户的请求）
$ sudo iptables -N syn-flood 

$ sudo iptables -A INPUT -p tcp --syn -j syn-flood 

$ sudo iptables -A syn-flood -m limit --limit 1/s --limit-burst 4 -j RETURN 

$ sudo iptables -A syn-flood -j DROP 
# 思路2：限制单个ip的最大syn连接数
$ sudo iptables –A INPUT –i eth0 –p tcp --syn -m connlimit --connlimit-above 15 -j DROP
```

防止DOS攻击
```sh
# 利用recent模块抵御DOS攻击
# 单个IP最多连接3个会话
$ sudo iptables -I INPUT -p tcp -dport 22 -m connlimit --connlimit-above 3 -j DROP 

# 只要是新的连接请求，就把它加入到SSH列表中
$ sudo iptables -I INPUT -p tcp --dport 22 -m state --state NEW -m recent --set --name SSH  

# 5分钟内你的尝试次数达到3次，就拒绝提供SSH列表中的这个IP服务。被限制5分钟后即可恢复访问。
$ sudo iptables -I INPUT -p tcp --dport 22 -m state NEW -m recent --update --seconds 300 --hitcount 3 --name SSH -j DROP 

```
防止单个ip访问量过大

```sh
$ sudo iptables -I INPUT -p tcp --dport 80 -m connlimit --connlimit-above 30 -j DROP 
```

木马反弹

```sh
$ sudo iptables –A OUTPUT –m state --state NEW –j DROP 
```

防止ping攻击

```sh
$ sudo iptables -A INPUT -p icmp --icmp-type echo-request -m limit --limit 1/m -j ACCEPT 
```


