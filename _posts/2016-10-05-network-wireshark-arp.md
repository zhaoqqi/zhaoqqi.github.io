---
layout: post
title: 使用Wireshark学习网络协议之ICMP
categories: [Network]
description: Learn icmp with Wireshark.
keywords: Network, ICMP
---

使用 ICMP 协议的两个目的：熟悉 Wireshark 的报文结构；该协议相对简单，完事开头难，先开个好头。

#### 什么是 ICMP

网络控制消息协议（英文：Internet Control Message Protocol，ICMP）是网路协议族的核心协议之一。它用于TCP/IP网络中发送控制消息，提供可能发生在通信环境中的各种问题反馈，通过这些信息，令管理者可以对所发生的问题作出诊断，然后采取适当的措施解决。

ICMP 依靠IP来完成它的任务，它是IP的主要部分。它与传输协议，如TCP和UDP显著不同：它一般不用于在两点间传输数据。它通常不由网络程序直接使用，除了ping和traceroute这两个特别的例子。 IPv4中的ICMP被称作ICMPv4，IPv6中的ICMP则被称作ICMPv6。

#### 技术细节

ICMP是在RFC 792中定义的互联网协议族之一。通常用于返回的错误信息或是分析路由。ICMP错误消息总是包括了源数据并返回给发送者。 ICMP错误消息的例子之一是TTL值过期。每个路由器在转发数据报的时候都会把IP包头中的TTL值减一。如果TTL值为0，“TTL在传输中过期”的消息将会回报给源地址。 每个ICMP消息都是直接封装在一个IP数据包中的，因此，和UDP一样，ICMP是不可靠的。

虽然ICMP是包含在IP数据包中的，但是对ICMP消息通常会特殊处理，会和一般IP数据包的处理不同，而不是作为IP的一个子协议来处理。在很多时候，需要去查看ICMP消息的内容，然后发送适当的错误消息到那个原来产生IP数据包的程序，即那个导致ICMP讯息被传送的IP数据包。

很多常用的工具是基于ICMP消息的。traceroute是通过发送包含有特殊的TTL的包，然后接收ICMP超时消息和目标不可达消息来实现的。ping则是用ICMP的"Echo request"（类别代码：8）和"Echo reply"（类别代码：0）消息来实现的。

#### ICMP 部分的报文结构

##### 报头

ICMP报头从IP报头的第160位开始（除非使用了IP报头的可选部分）。

| Bits  | 160-167  | 168-175 | 176-183 + 184-191 |
| ----- |:--------:| -------:|------------------:|
| 160   | Type     |   Code  | 校验码（checksum）|
| 192   | ID       |   ID    |   序号（sequence) |

* Type - ICMP的类型,标识生成的错误报文；
* Code - 进一步划分ICMP的类型,该字段用来查找产生错误的原因.；例如，ICMP的目标不可达类型可以把这个位设为1至15等来表示不同的意思。
* Checksum - 校验码部分,这个字段包含有从ICMP报头和数据部分计算得来的，用于检查错误的数据，其中此校验码字段的值视为0。
* ID - 这个字段包含了ID值，在Echo Reply类型的消息中要返回这个字段。
* Sequence - 这个字段包含一个序号，同样要在Echo Reply类型的消息中要返回这个字段。

##### 填充数据

填充的数据紧接在ICMP报头的后面（以8位为一组）：
* Linux的"ping"工具填充的ICMP除了8个8字节的报头以外，默认情况下还另外填充数据使得总大小为64字节。
* Windows的"ping.exe"填充的ICMP除了8个8字节的报头以外，默认情况下还另外填充数据使得总大小为40字节。

#### 使用 Wireshark 分析

##### 运行环境
```bash
测试机IP：192.168.1.103
目标机IP：192.168.1.1
测试命令：ping 192.168.1.1
```

##### 抓取 ICMP 数据包

截图：Wireshark抓到ping之后的主窗口截图

如上图所示，Wireshark 的主窗口分为三大块：Packlist List（数据包列表）、Packet Details（数据包细节）、Packet Bytes（数据包字节）。

数据包列表中的数据记录就是通过过滤器抓取到的所有数据包。单击其中一条数据包记录，可以在数据部细节中看到该数据包的详细信息，在数据部字节中看到16进制表示的数据包内容。

我们看下序列号为1的icmp的请求数据包的详细信息分为四行：

* > Frame 2: 74 bytes on wire (592 bits), 74 bytes captured (592 bits) on interface 0
* > Ethernet II, Src: ec:88:8f:74:b7:14 (ec:88:8f:74:b7:14), Dst: 60:d8:19:c7:a4:9b (60:d8:19:c7:a4:9b)
* > Internet Protocol Version 4, Src: 192.168.1.1 (192.168.1.1), Dst: 192.168.1.103 (192.168.1.103)
* > Internet Control Message Protocol

我们看到，`Packet Details`的内容会按照OSI模型中的层的概念划分，显示为以下内容：
* Frame：物理层的数据帧概况。
* Ethernet II：数据链路层以太网帧头部信息。
* Internet Protocol Version 4：互联网层IP包头部信息。
* Internet Control Message Protocol：传输层的数据段头部信息，此处是 ICMP 协议。

##### 分析 ICMP 数据包

###### 物理层的数据帧

```c
数据帧1：电缆上共74字节(592 bits)，网卡0上捕获74字节(592 bits)。
    网卡ID：0
    封装类型：Ethernet (1)
    到达时间：2016/10/04 06:57:10.029231000
    [本报文的时间偏移(time shift)：0.000000000]
    到达时刻：1475535430.029231000 seconds
    [Time delta from previous captured frame: 0.000000000 seconds]
    [Time delta from previous displayed frame: 0.000000000 seconds]
    [Time since reference or first frame: 0.000000000 seconds]
    帧编号：1
    帧长度：74字节(592 bits)
    捕获长度：74字节(592 bits)
    [帧是否被标记: False]
    [帧是否被忽略: False]
    [帧中的协议：eth:ethertype:ip:icmp:data]
    [使用颜色标记的rule name：ICMP]
    [使用颜色标记的rule string：icmp || icmpv6]
```

###### 数据链路层-以太网帧头部

```c
Ethernet II, 源：60:d8:19:c7:a4:9b(60:d8:19:c7:a4:9b)，目的：ec:88:8f:74:b7:14(ec:88:8f:74:b7:14)
    目的地：ec:88:8f:74:b7:14(ec:88:8f:74:b7:14)
        地址：ec:88:8f:74:b7:14(ec:88:8f:74:b7:14)
    来源：60:d8:19:c7:a4:9b(60:d8:19:c7:a4:9b)
        地址：60:d8:19:c7:a4:9b(60:d8:19:c7:a4:9b)
```

###### 网络层IP包头

```c
网络协议版本4，源：192.168.1.103(192.168.1.103)，目的：192.168.1.1(192.168.1.1)
    版本：4
    包头长度：20字节
    差分服务字段：0x00
    总长度：60
    标识：0x1e4b(7755)
    标志位：0x00
      0... = 保留位：未设置
      .0.. = Don't Fragment：未设置
      ..0. = More Fragment：未设置
    Fragment offset: 0
    Time to live: 64
    协议：ICMP(1)
    包头校验和：0xd8bd [validation disabled]
      [Good: False]
      [Bad: False]
    来源：192.168.1.103(192.168.1.103)
    目的：192.168.1.1(192.168.1.1)
    [来源GeoIP: 未知]
    [目的GeoIP: 未知]
```

###### IP 包中 ICMP 的内容

```c
/* 请求包 */
网络控制消息协议
    Type: 8 (Echo (ping) request)
    code: 0
    checksum: 0x4c5b [correct]
    Identifier (BE): 256 (0x0100)
    Identifier (LE): 1 (0x0001)
    Sequence number (BE): 1 (0x0001)
    Sequence number (LE): 256 (0x0100)
    [Response frame 2] 
    Data: (32 bytes)
      Data: 6162636465666768696a6b6c6d6e6f707172737475767761...
      Length: 32

/* 返回包 */
网络控制消息协议
    Type: 8 (Echo (ping) reply)
    code: 0
    checksum: 0x4c5b [correct]
    Identifier (BE): 256 (0x0100)
    Identifier (LE): 1 (0x0001)
    Sequence number (BE): 1 (0x0001)
    Sequence number (LE): 256 (0x0100)
    [Request frame 1]
    [Response time: 1.471 ms]
    Data: (32 bytes)
      Data: 6162636465666768696a6b6c6d6e6f707172737475767761...
      Length: 32
```
