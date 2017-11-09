---
layout:     post
title:      "study8.netstat.c源码分析"
date:       2017-11-8 11:00:00 
author:     "guanjunjian"
categories: 网络基础知识
tags:
    - network
    - study
---

* content
{:toc}

> Netstat 命令用于显示各种网络相关信息，如网络连接，路由表，接口状态 (Interface Statistics)，masquerade 连接，多播成员 (Multicast Memberships) 等等
>
>在本文中分析netstat.c就是想了解Netstat的实现原理
>
>[netstat.c源码](https://github.com/ecki/net-tools/blob/master/netstat.c) 




## 1. netstat的输出信息

执行netstat后，其输出结果为：

```
Active Internet connections (w/o servers)
Proto Recv-Q Send-Q Local Address Foreign Address State
tcp 0 2 210.34.6.89:telnet 210.34.6.96:2873 ESTABLISHED
tcp 296 0 210.34.6.89:1165 210.34.6.84:netbios-ssn ESTABLISHED
tcp 0 0 localhost.localdom:9001 localhost.localdom:1162 ESTABLISHED
tcp 0 0 localhost.localdom:1162 localhost.localdom:9001 ESTABLISHED
tcp 0 80 210.34.6.89:1161 210.34.6.10:netbios-ssn CLOSE

Active UNIX domain sockets (w/o servers)
Proto RefCnt Flags Type State I-Node Path
unix 1 [ ] STREAM CONNECTED 16178 @000000dd
unix 1 [ ] STREAM CONNECTED 16176 @000000dc
unix 9 [ ] DGRAM 5292 /dev/log
unix 1 [ ] STREAM CONNECTED 16182 @000000df
```


### 1.1 管理平面

管理平面是提供给网络管理人员使用TELNET、WEB、SSH、SNMP、RMON 等方式来管理设备，并支持、理解和执行管理人员对于网络设备各种网络协议的设置命令。管理平面提供了控制平面正常运行的前提，管理平面必须预先设置好控制平面中各种协议的相关参数，并支持在必要时刻对控制平面的运行进行干预。

### 1.2 控制平面

控制平面用于控制和管理所有网络协议的运行，例如生成树协议、VLAN 协议、ARP协议、各种路由协议和组播协议等等的管理和控制。控制平面通过网络协议提供给路由器/交换机对整个网络环境中网络设备、连接链路和交互协议的准确了解，并在网络状况发生改变时做出及时的调整以维护网络的正常运行。控制平面提供了数据平面数据处理转发前所必须的各种网络信息和转发查询表项。控制平面并不占用过多的硬件资源，但在正常状况下依然是网络设备CPU资源的主要占用平面，因此除了优化网络设备对于控制平面的调度流程和效率，一般还可以通过提供多CPU或提高CPU的处理性能来提高网络设备的控制平面性能。
  
<br/><br/>
控制平面主要靠CPU资源来处理信息。

show ip route 查看IP路由表，属控制平面范畴（路由信息数据库，RIB）



## 参考

* *[Linux netstat命令详解](https://www.cnblogs.com/ggjucheng/archive/2012/01/08/2316661.html)*
