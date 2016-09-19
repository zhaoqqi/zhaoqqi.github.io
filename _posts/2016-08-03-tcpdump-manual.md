---
layout: post
title: tcpdump常用方法手记
categories: [Network]
description: 工作中逐渐积累的 tcpdump 常用使用方法
keywords: tcpdump
---

就工作中经常用的抓包场景，记录一篇tcpdump的常用方法，随着遇到的场景变化逐渐补充。

##### 使用 [-i] 选项指定监听的网络接口：

```sh
# 监听运行设备(物理机/虚拟机)上所有进出 10.1.1.1 的包
tcpdump -i any host 10.1.1.1

# 监听运行设备(物理机/虚拟机)上网卡 eth0 所有进出 10.1.1.1 的包
tcpdump -i eth0 host 10.1.1.1

# 监听运行设备(物理机/虚拟机)上网桥 br_vm 所有进出 10.1.1.1 的包
tcpdump -i br_vm host 10.1.1.1

# 监听运行设备(物理机/虚拟机)上虚拟网卡 vnet0 所有进出 10.1.1.1 的包
tcpdump -i vnet0 host 10.1.1.1
```


##### 使用 [src/dst host] 指定流量方向

```sh
# 监听所有 10.1.1.1 发出的包
tcpdump -i any src host 10.1.1.1

# 监听所有 10.1.1.1 接收的包
tcpdump -i any dst host 10.1.1.1
```


##### 使用选项 [-vv] 输出详细的报文信息

```sh
# 主要用来查看报文的协议类型(tcp/udp...)
tcpdump -i any dst host 10.1.1.1 -vv
11:40:09.670182 IP (tos 0x0, ttl 108, id 5783, offset 0, flags [DF], proto TCP (6), length 40)
   $IP_1.12123 > $IP_2.58954: Flags [.], cksum 0x2233 (correct), seq 1, ack 1, win 65345, length 0
```

##### 使用选项 [-nn] 不进行端口名称的转换

```sh
# 这个没看出来作用……
tcpdump -i any dst host 10.1.1.1 -nn
11:43:47.829589 IP xxx.xx.xx.xxx_1.8360 > xx.xxx.xx.xx.44973: Flags [F.], seq 1, ack 1, win 122, options [nop,nop,TS val 691063383 ecr 319773296], length 0
11:43:47.833252 IP xx.xxx.xx.xx.44972 > xxx.xx.xx.xxx.8360: Flags [F.], seq 3525839232, ack 3704528140, win 4096, options [nop,nop,TS val 319773297 ecr 691032403], length 0
```

##### 组合使用 [-vv -nn]

```sh
# 公司网络运维的同学经常这么使用
tcpdump -i any dst host 10.1.1.1 -vv -nn
11:46:22.715211 IP (tos 0x0, ttl 214, id 3116, offset 0, flags [none], proto TCP (6), length 52)
    xxx.xx.xx.xxx.2847 > xx.xxx.xx.xx.46077: Flags [.], cksum 0x9320 (correct), seq 4, ack 28, win 65211, options [nop,nop,TS val 863567 ecr 691217938], length 0
```

##### 监听指定端口 [and port $PORT]

```sh
tcpdump -i br_vm dst host 10.1.1.1 and port 22 -vv -nn

tcpdump: WARNING: br_vm: no IPv4 address assigned
tcpdump: listening on br_vm, link-type EN10MB (Ethernet), capture size 65535 bytes

15:11:52.498597 IP (tos 0x0, ttl 46, id 11422, offset 0, flags [DF], proto TCP (6), length 52)
    114.255.44.132.22237 > 10.1.1.1.22: Flags [S], cksum 0xd7b7 (correct), seq 2879601682, win 64240, options [mss 1460,nop,wscale 0,nop,nop,sackOK], length 0
15:11:53.010222 IP (tos 0x0, ttl 46, id 11434, offset 0, flags [DF], proto TCP (6), length 52)
    114.255.44.132.22237 > 10.1.1.1.22: Flags [S], cksum 0xd7b7 (correct), seq 2879601682, win 64240, options [mss 1460,nop,wscale 0,nop,nop,sackOK], length 0
15:11:53.524167 IP (tos 0x0, ttl 46, id 11436, offset 0, flags [DF], proto TCP (6), length 48)
    114.255.44.132.22237 > 10.1.1.1.22: Flags [S], cksum 0xebbe (correct), seq 2879601682, win 64240, options [mss 1460,nop,nop,sackOK], length 0

```

##### 持续补充中……
