---
layout:     post
title:      "study12.net_cls文档--------net_cls.txt「译」"
date:       2017-11-28 15:00:00 
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
> [Documentation/cgroups/net_cls.txt](https://www.mjmwired.net/kernel/Documentation/cgroups/net_cls.txt)的翻译。
>




## Network classifier cgroup

-----------------------------------

Network classifier cgroup提供一个接口，这个接口可以使用class id来标记（tag）网络数据包。

<br/>
TC可以使用这个标记来分配不同的优先级给不同的cgroup。Netfilter（iptables）也可以使用这个标记对数据包施行action。

<br/>
创建一个net_cls实例需要创建net_cls.classid文件。
当net_cls.classid中的值为0时，该文件无效。

<br/>
你可以写入十六进制的值到net_cls.classid文件中；这些值的格式为`0xAAAABBBB`；`AAAA`是主句柄号而`BBBB`是副句柄号。
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

读取时会显示：
```
cat /sys/fs/cgroup/net_cls/0/net_cls.classid
1048577
```

配置tc，使用tc创建一个基于classid 10：1的filter限制其速率为40mbit：

```
tc qdisc add dev eth0 root handle 10: htb

tc class add dev eth0 parent 10: classid 10:1 htb rate 40mbit 
  - creating traffic class 10:1

tc filter add dev eth0 parent 10: protocol ip prio 10 handle 1: cgroup
```

配置iptables，最基础的例子：

```
iptables -A OUTPUT -m cgroup ! --cgroup 0x100001 -j DROP
```


## 参考

* *[Hierachical token bucket theory](http://luxik.cdi.cz/~devik/qos/htb/manual/theory.htm)*

