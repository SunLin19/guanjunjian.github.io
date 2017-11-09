---
layout:     post
title:      "study8./proc/net/tcp分析"
date:       2017-11-9 11:00:00 
author:     "guanjunjian"
categories: 网络基础知识
tags:
    - network
    - study
---

* content
{:toc}

> /proc/net/tcp文件提供了tcp的连接信息，是由net/ipv4/tcp_ipv4.c中的tcp4_seq_show()实现信息打印的
>
>本文内容来源于[linux官方文档proc_net_tcp.txt](https://github.com/torvalds/linux/blob/v4.10/Documentation/networking/proc_net_tcp.txt)
>




## 1. 官方文档解释

proc_net_tcp.txt介绍了/proc/net/tcp和/proc/net/tcp6接口。这些接口展示了tcp的连接信息。

展示的信息中，首先是listening TCP sockets，接着是established TCP sockets。

在linux中执行cat /proc/net/tcp，输出结果如图1：

![](/img/study/study-8-proc-net-tcp-analysis/img-1-execute-result.png)

文档中对其中每个字段的解释如下（由于长度原因，分为3个部分）：

**part1:**

```
   46: 010310AC:9C4C 030310AC:1770 01 
   |      |      |      |      |   |--> connection state（套接字状态）
   |      |      |      |      |------> remote TCP port number（远端端口，主机字节序）
   |      |      |      |-------------> remote IPv4 address（远端IP，网络字节序）
   |      |      |--------------------> local TCP port number（本地端口，主机字节序）
   |      |---------------------------> local IPv4 address（本地IP，网络字节序）
   |----------------------------------> number of entry
```

`connection state`(套接字状态)，不同的数值代表不同的状态，参照如下：

```
TCP_ESTABLISHED:1   TCP_SYN_SENT:2
TCP_SYN_RECV:3      TCP_FIN_WAIT1:4
TCP_FIN_WAIT2:5     TCP_TIME_WAIT:6
TCP_CLOSE:7         TCP_CLOSE_WAIT:8
TCP_LAST_ACL:9      TCP_LISTEN:10
TCP_CLOSING:11
```

**part2:**

```
   00000150:00000000 01:00000019 00000000  
      |        |     |     |       |--> number of unrecovered RTO timeouts（超时重传次数）
      |        |     |     |----------> number of jiffies until timer expires（超时时间，单位是jiffies）
      |        |     |----------------> timer_active (定时器类型，see below)
      |        |----------------------> receive-queue（根据状态不同有不同表示,see below）
      |-------------------------------> transmit-queue(发送队列中数据长度)
```

`receive-queue`，当状态是ESTABLISHED，表示接收队列中数据长度；状态是LISTEN，表示已经完成连接队列的长度。

`timer_active`:

```
  0  no timer is pending  //没有启动定时器
  1  retransmit-timer is pending  //重传定时器
  2  another timer (e.g. delayed ack or keepalive) is pending  //连接定时器、FIN_WAIT_2定时器或TCP保活定时器
  3  this is a socket in TIME_WAIT state. Not all fields will contain data (or even exist)  //TIME_WAIT定时器
  4  zero window probe timer is pending  //持续定时器
```

**part3:**

```
   1000        0 54165785 4 cd1e6040 25 4 27 3 -1
    |          |    |     |    |     |  | |  | |--> slow start size threshold, or -1 if the threshold is >= 0xFFFF
    |          |    |     |    |     |  | |  |      （如果慢启动阈值大于等于0xFFFF则显示-1，否则表示慢启动阈值）
    |          |    |     |    |     |  | |  |      
    |          |    |     |    |     |  | |  |----> sending congestion window（当前拥塞窗口大小）
    |          |    |     |    |     |  | |-------> (ack.quick<<1)|ack.pingpong（快速确认数和是否启用的标志位的或运算结果）
    |          |    |     |    |     |  |---------> Predicted tick of soft clock (delayed ACK control data)（用来计算延时确认的估值）
    |          |    |     |    |     |             
    |          |    |     |    |     |------------> retransmit timeout（）（RTO，单位是clock_t）
    |          |    |     |    |------------------> location of socket in memory（socket实例的地址）
    |          |    |     |-----------------------> socket reference count（socket结构体的引用数）
    |          |    |-----------------------------> inode（套接字对应的inode）
    |          |----------------------------------> unanswered 0-window probes（see below）
    |---------------------------------------------> uid（用户id）
```

`unanswered 0-window probes`：持续定时器或保活定时器周期性发送出去但未被确认的TCP段数目，在收到ACK之后清零

### 1.1 管理平面

管理平面是提供给网络管理人员使用TELNET、WEB、SSH、SNMP、RMON 等方式来管理设备，并支持、理解和执行管理人员对于网络设备各种网络协议的设置命令。管理平面提供了控制平面正常运行的前提，管理平面必须预先设置好控制平面中各种协议的相关参数，并支持在必要时刻对控制平面的运行进行干预。

### 1.2 控制平面

控制平面用于控制和管理所有网络协议的运行，例如生成树协议、VLAN 协议、ARP协议、各种路由协议和组播协议等等的管理和控制。控制平面通过网络协议提供给路由器/交换机对整个网络环境中网络设备、连接链路和交互协议的准确了解，并在网络状况发生改变时做出及时的调整以维护网络的正常运行。控制平面提供了数据平面数据处理转发前所必须的各种网络信息和转发查询表项。控制平面并不占用过多的硬件资源，但在正常状况下依然是网络设备CPU资源的主要占用平面，因此除了优化网络设备对于控制平面的调度流程和效率，一般还可以通过提供多CPU或提高CPU的处理性能来提高网络设备的控制平面性能。
  
<br/><br/>
控制平面主要靠CPU资源来处理信息。

show ip route 查看IP路由表，属控制平面范畴（路由信息数据库，RIB）



## 参考

* *[/proc/net/tcp中各项参数说明](http://blog.csdn.net/justlinux2010/article/details/21028797)*
