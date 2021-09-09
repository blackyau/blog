---
layout: post
title: 主机互 ping 交互过程
date: 2021-09-09 23:26:28
updated: 2021-09-09 23:26:31
categories: 学习笔记
tags: 
    - 计算机网络
    - 路由器
    - ARP
    - ICMP
urlname: 27
comment: true
---

本文从一个 ping 命令作为一个起点，详细描述了数据在路由器之间流通的路径，以及 ARP 请求和回复的详细过程，还有数据在网络层、链路层之间封装和传递的过程。

<!-- more -->

拓扑结构如下

![image.png](https://st.blackyau.net/blog/27/1.png)

# 1.R2(模拟PC1) 发送ping

## 1.2 应用层

应用程序要求使用 ICMP 协议发送一个给 192.168.1.2 类型为 8 的回显请求

## 1.3 网络层

通过向 192.168.1.2 发送一个 ICMPv4 数据报，网络层尝试向远程主机发送一个请求，源 IP 为 192.168.3.2，目标 IP 为 192.168.1.2。其中 ICMP 数据中的类型字段为 8 代表这是一个 ICMPv4 的回显请求。IP 首部的协议为 1 即 ICMP。

### 1.3.1 路由选择

Pre: 优先级，优先级越高取值越小
```
<R2>dis ip rou	
Route Flags: R - relay, D - download to fib
------------------------------------------------------------------------------
Routing Tables: Public
         Destinations : 5        Routes : 5        

Destination/Mask    Proto   Pre  Cost      Flags NextHop         Interface

        0.0.0.0/0   Static  60   0          RD   192.168.3.1     GigabitEthernet0/0/0
      127.0.0.0/8   Direct  0    0           D   127.0.0.1       InLoopBack0
      127.0.0.1/32  Direct  0    0           D   127.0.0.1       InLoopBack0
    192.168.3.0/24  Direct  0    0           D   192.168.3.2     GigabitEthernet0/0/0
    192.168.3.2/32  Direct  0    0           D   127.0.0.1       GigabitEthernet0/0/0
```
当 TCP/IP 需要向某个 IP 地址发起通信时，它会对路由表进行评估，以确定如何发送数据包。

TCP/IP 使用需要通信的目的 IP 地址和路由表中每一个路由项的掩码进行按位与运算，如果与运算后的结果，匹配对应路由项的网络地址，则记录下次路由项。当计算完路由表中所有的路由项后。

- 会使用最长匹配路由（掩码中具有最多 1 的路由项）来和此目的 IP 地址进行通信
- 如果存在多个最长匹配路由，那么选择具有最高优先级（值最小）的路由表
- 如果存在多个具有最高优先级的最长匹配路由，那么选择最开始找到的最长匹配路由（如果是其它设备的话，一般会看接入方式，有线>无线>移动信号4G）

在这里它就选择了 `0.0.0.0/0` 的默认路由，决定了会把该数据包交给 `192.168.3.1` 帮忙传下去。

## 1.4 ARP

```
<R2>dis arp
IP ADDRESS      MAC ADDRESS     EXPIRE(M) TYPE        INTERFACE   VPN-INSTANCE VLAN/CEVLAN PVC                      
------------------------------------------------------------------------------
192.168.3.2     5489-98da-3e52            I -         GE0/0/0
------------------------------------------------------------------------------
Total:1         Dynamic:0       Static:0     Interface:1
```

通过上面的路由选择过程后，确认了该数据包的下一跳为 `192.168.3.1` 但是在交给链路层包装以太帧之前，会先使用 ARP 用于确定目标 IP 对应的 MAC 地址。

查询 ARP 表后，发现 ARP 表中没有与之相对应的项目，所以就需要发起 ARP 请求。寻找 `192.168.3.1` 的 MAC 地址。

### 1.4.1 发送 ARP 请求

![image.png](https://st.blackyau.net/blog/27/2.png)

ARP 向 192.168.3.0/24 广播 ARP 请求，其中源 MAC 地址和源 IP 地址都是自己对应接口的，目标 IP 就分别是 192.168.3.2，同时目标 MAC 为全 0。

链路层封装该 ARP 数据包时，使用源地址为端口的 MAC 地址，目标地址使用全 1，并将其中的类型字段设置为 0x0806 也就是 ARP 对应的值。

### 1.4.2 收到 ARP 请求

R1 收到该 ARP 请求后，发现该请求的目标 IP 是自己。它首先会更新自己的 ARP 表，将该请求的源 IP 和源 MAC 地址写入 ARP 表中。这样可以减少一次 ARP 请求，提升效率。

```
<R1>dis arp
IP ADDRESS      MAC ADDRESS     EXPIRE(M) TYPE        INTERFACE   VPN-INSTANCE VLAN/CEVLAN PVC                      
------------------------------------------------------------------------------
192.168.0.1     5489-989b-51f2            I -         Eth0/0/0
192.168.1.1     5489-989b-51f3            I -         Eth0/0/1
192.168.1.2     5489-982b-3b01  20        D-0         Eth0/0/1
192.168.3.1     5489-989b-51f4            I -         GE0/0/0
192.168.3.2     5489-98da-3e52  20        D-0         GE0/0/0  // 新写入的条目
------------------------------------------------------------------------------
Total:5         Dynamic:2       Static:0     Interface:3    
```

### 1.4.2 回复 ARP 请求
![image.png](https://st.blackyau.net/blog/27/3.png)

R1 收到该 ARP 请求后，发现该请求的目标 IP 是自己，然后就会调换源地址和目标地址，并且将自己的 MAC 地址写入源地址的字段中，发送该 ARP 回复。

链路层封装该 ARP 数据包时，使用源地址为自己的 MAC 地址，目标地址为发送该请求的 MAC 地址。

### 1.4.3 收到 ARP 回复

```
<R2>dis arp
IP ADDRESS      MAC ADDRESS     EXPIRE(M) TYPE        INTERFACE   VPN-INSTANCE VLAN/CEVLAN PVC                      
------------------------------------------------------------------------------
192.168.3.2     5489-98da-3e52            I -         GE0/0/0
192.168.3.1     5489-989b-51f4  20        D-0         GE0/0/0
------------------------------------------------------------------------------
Total:2         Dynamic:1       Static:0     Interface:1    
```

R2 将回复的 MAC 地址写入自己的 ARP 缓存中。

然后 R2 就继续发送引起这次 ARP 请求/应答交换过程的数据报。把数据转发出去的时候，还会将 TTL - 1。

## 1.5 链路层

![image.png](https://st.blackyau.net/blog/27/4.png)

收到网络层的 ICMP 数据包，将其包装为以太网帧，其中源地址为 R2 的 MAC 地址，目标地址为刚刚通过 ARP 请求得到的 MAC 地址。

# 2. R1 转发数据包

路由器收到了由 192.168.3.2 发来 ICMP 数据帧，对其进行解析。

## 2.1 网络层

```
[Huawei]dis ip rou
Route Flags: R - relay, D - download to fib
------------------------------------------------------------------------------
Routing Tables: Public
         Destinations : 8        Routes : 8        

Destination/Mask    Proto   Pre  Cost      Flags NextHop         Interface

      127.0.0.0/8   Direct  0    0           D   127.0.0.1       InLoopBack0
      127.0.0.1/32  Direct  0    0           D   127.0.0.1       InLoopBack0
    192.168.0.0/24  Direct  0    0           D   192.168.0.1     Ethernet0/0/0
    192.168.0.1/32  Direct  0    0           D   127.0.0.1       Ethernet0/0/0
    192.168.1.0/24  Direct  0    0           D   192.168.1.1     Ethernet0/0/1
    192.168.1.1/32  Direct  0    0           D   127.0.0.1       Ethernet0/0/1
    192.168.3.0/24  Direct  0    0           D   192.168.3.1     GigabitEthernet0/0/0
    192.168.3.1/32  Direct  0    0           D   127.0.0.1       GigabitEthernet0/0/0
```

R1 查询自己的路由表，发现目的 IP 192.168.1.2 有匹配的项目，所以该数据包会向 Ethernet0/0/1 发送。

## 2.2 ARP

路由器查询自己的 ARP 表，源 IP 的 MAC 地址已存在与 ARP 表中，因为之前回复它的 ARP 请求的时候，也在自己的 ARP 表中存了一份。

目的 IP 192.168.3.2 的 MAC 不在 ARP 表中，所以和上一次一样也需要经过 ARP 请求/回复获取其 MAC 写入 ARP 表，再继续执行后面操作。

## 2.3 链路层

收到网络层的 ICMP 数据包，将其包装为以太网帧，其中源地址为 R1 的 MAC 地址，目标地址为刚刚通过 ARP 请求得到的 MAC 地址。

# 3. PC2 回复ping

![image.png](https://st.blackyau.net/blog/27/5.png)

收到来自 192.168.3.2 发送的 ICMPv4 请求，而且目标 IP 是自己，所以回复该 ICMP 请求。

## 3.1 网络层

将接收到的数据返回给发送者，调换源 IP 和目的 IP，IGMP 数据中的类型为 0 代表这是一个回复。

发送回复的时候，还是和之前一样，要先查自己的路由表。然后因为路由表中的默认路由匹配了，所以就会将该数据包交给默认网关处理。

## 3.2 ARP

再次通过 ARP 表，找到默认网关 192.168.1.1 的 MAC 地址。

## 3.3 链路层

封装以太网帧的时候，使用目的地址为默认网关的 MAC 地址。

# 4. R1 转发数据包

- 查询路由表，发现目的 IP 在路由表中
- 查询 ARP 表，找到目的 IP 的 MAC 地址
- 链路层封装数据帧，发送给 R2

# 5. R2 收到应答

收到回复后，将当前时间与应答中的时间相减，也就获得了到达被 ping 主机的 RTT 估计值。

对于 ICMPv4 来说，类型为 0/8 的应答与请求，时间戳为可选的字段。而类型为 14/13 的应答和请求就需要带时间戳，但是现在已经作废不用。

# 后续

当 PC1 继续发送 ICMP 包的话，他会将序列号 +1，然后继续发送。此时因为 ARP 缓存都已存在，所以不会需要 ARP 请求/回复，链路层会直接调用 ARP 缓存中的 MAC 地址进行发送。

# 参考

- [华为文档中心 路由协议基础](https://support.huawei.com/enterprise/zh/doc/EDOC1100087024)
- [TCP/IP详解 卷1：协议（原书第2版）](https://item.jd.com/11966296.html)
- [CSDN@--Allen-- 96-ICMP 协议（时间戳请求与应答）](https://blog.csdn.net/q1007729991/article/details/72600130)
