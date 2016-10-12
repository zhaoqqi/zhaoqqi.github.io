---
layout: post
title: Wireshark使用手册
categories: [Network]
description: Wireshark基本使用方法记录.
keywords: Network
---

#### 是什么

Wireshark 是可以根据协议类型、源或目的主机、目标端口不同，对网络数据包进行抓取和分析的强有力的工具软件。

#### 作用

本篇作为整理网络知识体系的准备篇，需要将 Wireshark 作为分析各种网络协议的得力工具，贯穿整个网络系列学习文章书写的始终。

#### 过滤器

使用 Wireshark 的默认设置时，会抓取到大量冗余的数据包，以至于很难找到自己需要的部分。这时候就需要使用过滤器。

Wireshark 的过滤器分为两种类型：捕获过滤器 和 显示过滤器。

##### 捕获过滤器

捕获过滤器是数据经过第一次过滤，通过网络协议类型、源主机、目的主机、端口号等参数，来过滤你需要的数据包，语法有点类似 tcpdump。捕捉过滤器必须在开始捕捉前设置完毕，这一点跟显示过滤器是不同的。 

设置使用那个 filter 进行过滤：
1. 选择 Capture -> Options ；
2. 在 Capture Filters 栏填写过滤规则或者点击该栏；
3. 点击后选择之前保持好的过滤器；
4. 点击 start 开始过滤数据包；

新增一个捕获过滤器：
1. 选择 Capture -> Options ；
2. 点击 Capture Filters -> New；
3. 填写 Filter name 和 Filter string；
4. 保持

```bash
语法：Protocol  Direction  Host(s)  Value  Logical Operations  Other expression
例子：tcp       dst        10.1.1.1  80           and          tcp dst 10.2.2.2 3128
```

###### Protocol（协议）
* 可能的值: ether, fddi, ip, arp, rarp, decnet, lat, sca, moprc, mopdl, tcp and udp。
* 如果没有特别指明是什么协议，则默认使用所有支持的协议。

###### Direction（方向）
* 可能的值: src, dst, src and dst, src or dst。
* 如果没有特别指明方向，则默认使用 `src or dst` 作为关键字。

###### Host(s)
* 可能的值： net, port, host, portrange。
* 如果没有指定，则默认使用"host"关键字。

###### Logical Operations（逻辑运算）
* 可能的值：not, and, or。
* 否("not")具有最高的优先级。或("or")和与("and")具有相同的优先级，运算时从左至右进行。

###### 例子
```bash
# 显示目的TCP端口为8898的封包
tcp dst port 8898

# 显示来自IP 192.168.1.2的封包
ip src host 192.168.1.2

# 显示来源或者目的为192.168.1.2的封包
host 192.168.1.2

# 显示来源是udp或者tcp，并且端口号在21-25范围内的封包
src portrange 21-25

# 显示出了icmp以外的封包
not icmp

# 显示来源是10.7.2.12，并且目的不是10.200.0.0/16的封包
src host 10.7.2.12 and not dst net 10.200.0.0/16

# 显示来源是10.4.1.12或者10.6.0.0/16，目的tcp端口在200-10000范围内，并且目的不是10.0.0.0/8的封包
(src host 10.4.1.12 or src net 10.6.0.0/16) and tcp dst portrange 200-10000 and dst net 10.0.0.0/8
```


##### 显示过滤器

通常经过铺货过滤器后的数据还是很复杂，显示过滤器的过滤功能更加强大，它允许你在日志中迅速准确的找到所需要的记录。它的功能比捕捉过滤器更为强大，而且在您想修改过滤器条件时，并不需要重新捕捉一次。

```bash
语法：Protocol.string1.string2 | Comparison operator | Value | Logical Operations | Other expression
例子：ftp      passive    ip              ==          10.2.3.4       xor                  icmp.type
```

###### Protocol（协议）
你可以使用大量位于OSI模型第2至7层的协议。点击"Expression..."按钮后，你可以看到它们。比如：IP，TCP，DNS，SSH。

###### String1, String2 (可选项)
协议的子类，点开协议名称左侧的 + 号可以看到它们。

###### Comparison operators（比较运算符）
```bash
== 
!=
>
<
>=
<=
contains
matches
```

###### Logical expressions（逻辑运算符）
```bash
and &&
or  ||
xor ^^ 只有当且仅当其中一个条件满足时为真
not !
```

###### 例子
```
# 显示snmp或dns或icmp的封包
snmp || dns || icmp 

# 显示原或者目的ip地址为192.168.1.1的封包
ip.addr == 192.168.1.1

# 显示来源IP地址除了10.1.2.3，或者目的IP地址除了10.4.5.6的封包
ip.src != 10.1.2.3 or ip.dst != 10.4.5.6

# 显示来源IP地址除了10.1.2.3，并且目的IP地址除了10.4.5.6的封包
ip.src != 10.1.2.3 and ip.dst != 10.4.5.6

# 显示来源或者目的TCP端口为25的封包
tcp.port == 25

# 显示目的TCP端口为25的封包
tcp.dst_port == 25

# 显示带有TCP标识的封包
tcp.flags 

# 显示带有TCP SYNC的封包
tcp.flags.syn == 0x02
```

如果过滤器的语法正确，表达式的背景显示为绿色；如果过滤器语法错误，背景显示为红色。
