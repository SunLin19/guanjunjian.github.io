---
layout:     post
title:      "study16.[net_cls]cls_cgroup_classify()分析"
date:       2017-12-14 15:00:00
author:     "guanjunjian"
categories: 网络流量控制
tags:
    - network
    - cgroup
    - net_cls
    - study
---

* content
{:toc}

>
> 了解该函数的目的是根据[net_next](https://lists.linuxfoundation.org/pipermail/containers/2014-January/033844.html)
>
> 基于内核4.3，文档生成时间 2015-11-02 12:44 EST。
>
> 这个子系统使用等级识别符（classid）标记网络数据包，可允许 Linux 流量控制程序（tc）识别从具体 cgroup 中生成的数据包。
>




## Network priority cgroup

-----------------------------------

Network classifier cgroup提供一个接口，这个接口可以使用class id来标记（tag）网络数据包。

<br/>
TC可以使用这个标记来分配不同的优先级给不同的cgroup。Netfilter（iptables）也可以使用这个标记对数据包施行action。

<br/>
创建一个net_cls实例需要创建net_cls.classid文件。
当net_cls.classid中的值为0时，该文件无效。

<br/>
你可以写入十六进制的值到net_cls.classid文件中；这些值的格式为`0xAAAABBBB`；`AAAA`是主句柄号而`BBBB`是副句柄号。(如果没有就写0，并且0是可以省略的,0x10001=0x0000100001=0x1:1)
读取net_cls.classid时会显示十进制结果。

<br/>
一个例子：

```
mkdir /sys/fs/cgroup/net_cls
mount -t cgroup -onet_cls net_cls /sys/fs/cgroup/net_cls
mkdir /sys/fs/cgroup/net_cls/0
//设置classid，标记group组的网络包标记为10：1，- setting a 10:1 handle.
echo 0x100001 >  /sys/fs/cgroup/net_cls/0/net_cls.classid  		
```

<br/>
读取时会显示：
```
cat /sys/fs/cgroup/net_cls/0/net_cls.classid
1048577
```

<br/>
配置tc，使用tc创建一个基于classid 10：1的filter限制其速率为40mbit：

```
tc qdisc add dev eth0 root handle 10: htb

tc class add dev eth0 parent 10: classid 10:1 htb rate 40mbit
  - creating traffic class 10:1

tc filter add dev eth0 parent 10: protocol ip prio 10 handle 1: cgroup
```

<br/>
配置iptables，最基础的例子：

```
iptables -A OUTPUT -m cgroup ! --cgroup 0x100001 -j DROP
```


## 参考

* *[Documentation/cgroups/net_cls.txt](https://www.mjmwired.net/kernel/Documentation/cgroups/net_cls.txt)*
* *[Cgroup相关介绍](http://www.aboutyun.com/thread-5891-1-1.html)*
