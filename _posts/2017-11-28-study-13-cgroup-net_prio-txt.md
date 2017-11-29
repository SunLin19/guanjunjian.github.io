---
layout:     post
title:      "study13.net_prio文档--------net_prio.txt「译」"
date:       2017-11-28 17:00:00 
author:     "guanjunjian"
categories: 网络流量控制
tags:
    - network
    - cgroup
    - net_prio
    - study
---

* content
{:toc}

>
> [Documentation/cgroups/net_prio.txt](https://www.mjmwired.net/kernel/Documentation/cgroups/net_prio.txt)的翻译。
>
> 基于内核4.3，文档生成时间 2015-11-02 12:44 EST。
> 
> 这个子系统提供了一种动态控制每个网卡流量优先级的功能
>




## Network priority cgroup

-----------------------------------

Network priority cgroup提供了一个允许管理者动态设置由应用产生的网络流量优先级的接口。

表面上，一个应用可以使用socket的选项SO_PRIORITY来设置它产生的流量的优先级，然而，这并非总是可能的，因为：

* 1) 应用程序可能没有在代码中设置这个值
* 2) 应用流量的优先级常常是特定地点的管理决定(site-specific administrative decision)而不是由应用程序自身定义的

该cgroup运行管理者将一个进程分配到一个cgroup中，在这个cgroup中定义了在指定网络接口中，外出流量（egress traffic）的优先级。Network priority group可以在第一次挂载cgroup文件系统的时候创建。

```
# mount -t cgroup -onet_prio none /sys/fs/cgroup/net_prio
```

经过上面步骤后，最初的cgroup作为父cgroup，可在`/sys/fs/cgroup/net_prio`中看到，父cgroup包含了系统中的所有task。`/sys/fs/cgroup/net_prio/tasks`列出该cgroup中的所有task。

每个net_prio cgroup包含由子系统特定的两个文件：

**1) net_prio.prioidx**

该文件是只读的，作为简单的信息展示。它包含一个唯一的数值，在内核内部中，内核使用该数值来代表该cgroup。

**2) net_prio.ifpriomap**

该文件中包含一个map，该map是该cgroup中进程从各个网络接口向往流出流量的优先级映射。它包含了一系列以`<ifname priority>`形式的数组(tuples)。该文件的内容可以使用`echo <ifname priority>`进行修改。例如：

```
echo "eth0 5" > /sys/fs/cgroups/net_prio/iscsi/net_prio.ifpriomap
```

以上命令将使得iscsi net_prio cgroup中的进程产生的向eth0外出的流量拥有优先级5。父cgroup中也有一个可写的`net_prio.ifpriomap`文件，该文件用来设置整个系统的默认优先级。

该文中的`优先级`是对在排队传输到队列规则(qdisc)的数据帧设置的，该优先级是立即生效的。因此优先级将在硬件队列做出选择之前进行分配。

net_prio的用处之一是，使用mqprio qdisc时，运行应用程序流量基于流量类型流向硬件/驱动程序。上文提到的map可以被管理者或别的网络协议例如DCBX使用。

一个新创建的net_prio cgroup将继承父cgroup的配置。


## 参考

* *[Documentation/cgroups/net_prio.txt](https://www.mjmwired.net/kernel/Documentation/cgroups/net_prio.txt)*
* *[Cgroup相关介绍](http://www.aboutyun.com/thread-5891-1-1.html)*
