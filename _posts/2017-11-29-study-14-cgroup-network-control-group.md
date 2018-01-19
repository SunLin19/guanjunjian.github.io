---
layout:     post
title:      "「十四」利用Cgroup限制网络带宽"
date:       2017-11-29 11:00:00 
author:     "guanjunjian"
categories: 网络流量控制
tags:
    - network
    - cgroup
    - net_cls
    - htb
    - study
---

* content
{:toc}

>
> 本文是对[CONTROLLING NETWORK RESOURCES USING CONTROL GROUPS](http://vger.kernel.org/netconf2009_slides/Network%20Control%20Group%20Whitepaper.odt)的翻译。
> 
> cgroup net_cls与tc htb结合使用，达到对cgroup网络资源限制的目的。
>

## 简介

Control Groups为聚合/分区task集提供了一种机制，而且这些task的子task也会进入相同的层级的cgroup。cgroup的网络子系统可以提供资源控制从而达到调度资源或实施每个cgroup限制的目的。

这篇文档将解释cgroup在Linux数据包调度环境下的架构，并且将给出可亲手实践的例子来解释如何使用它。

## 架构

使用cgroup控制网络资源的基本思想是将cgroup与已经存在的能提供分类和调度网络数据包功能的网络数据包分类器和调度框架连接起来。

为了达到这一的目的，创建了一种新的cgroup子系统net_cls,该子系统可以让cgroup可以分辨数据包流量是从哪个cgroup中流出的。这是通过给cgroup分配一个classid实现的，classid可以被数据包分类器cls_cgroup使用，从而将数据包过滤到classid匹配的流量类型中，如下图1.所示。

图1：

![](/img/study/study-14-cgroup-network-control-group/1-architecture.png)




为了尽可能保持架构的简单和非侵入性，当数据包在网络栈中传递的时候，没有将classid存储在数据包中。数据包分类器使用数据包的进程上下文信息（process context information）来查找classid。这样做的缺点是：这种方法是适合在进程上下文中离开网络栈的数据包，例如，这种方法不适合内核生成的数据包（ACKs，ICMP replies等等）或者不适合于数据包在经过数据包分类器之前就进行了排队和重调度的数据包。

流量分类(traffic class)可以以树的形式来组织，也可以包含任何数量的队列规则(qdisc)来实现优先级、带宽限制或公平队列。流量分类必须分配到网络接口(network interface)，因此，如果一个cgroup发送数据到多个网络接口，就需要为每一个网络接口维护一个流量类型。

## 必要条件

内核和用户模式补丁都必须升级到适合RHEL6的级别，该功能是默认开启的，内核IDE不需要特殊配置。确切的版本要求是：Linux kernel >= 2.6.29和iproute2 >= 2.6.30

## 配置

第一步是，创建cgroup设备目录，通过挂载网络分类子系统（net_cls）到虚拟文件系统（VFS）中来达到装载网络分类子系统的目的。然后，通过新建目录达到新建cgroup的目的。可以通过参考cgroup文档来获得关于如何挂载和新建cgroup的详细信息。

```
# mkdir -p /dev/cgroup
# mount -t cgroup net_cls -o net_cls /dev/cgroup
# mkdir /dev/cgroup/A
# mkdir /dev/cgroup/B
```

为了给cgroup分配classid，可以将classid写入对应cgroup目录下的net_cls.classid文件。

```
# cd /dev/cgroup
# echo 0x1001 > A/net_cls.classid
# 10:1
# echo 0x1002 > B/net_cls.classid
# 10.2
```
以上的例子将会指导数据包分类器将来自cgroup A的数据包过滤到流量类型10:1中，将来自cgroup B的数据包过滤到流量类型10:2中。

最后，以下的例子使用简单的分层令牌桶排队规则（HTB）说明简单的使用。如何调度数据包不是该文档的阐述内容，可以参考其他适合的文档，例如如何配置复杂的数据包调度来达到流量分类之间带宽的共享的文档。

```
tc qdisc add dev eth0 root handle 10: htb
```

以上命令给网络接口eth0添加了HTB队列规则，该规则可以与类型和过滤器挂钩。该队列规则使用标识句柄`10: `，所以所有子类型都必须以`10:X`命名。

新添加的队列规则被当做父类，可以为每个cgroup添加流量类型子类。注意，classid 10:1和10:2与net_cls.classid文件中的classid匹配。在这个例子中，每个流量类型都添加了带宽限制。

最后但不是最重要的，真正的分类器需要添加到队列规则上，从而将数据包过滤到正确的流量类型中。

```
# tc filter add dev eth0 parent 10: protocol ip prio 10 handle 1: cgroup
```

## 参考
* Linux-Net Wiki http://www.linuxfoundation.org/en/Net
* Linux Advanced Routing & Traffic Control HOWTO http://lartc.org/howto/
* Control Groups Documentation linux/Documentation/cgroups/

## 我的参考

* *[CONTROLLING NETWORK RESOURCES USING CONTROL GROUPS](http://vger.kernel.org/netconf2009_slides/Network%20Control%20Group%20Whitepaper.odt)*
* *[限制单个进程的带宽](https://www.topjishu.com/3186.html)*

