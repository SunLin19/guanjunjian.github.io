---
layout:     post
title:      "study15.博文《再谈Docker容器单机网络：利用iptables trace和ebtables log》读后整理"
date:       2017-12-5 11:00:00 
author:     "guanjunjian"
categories: 容器网络知识
tags:
    - network
    - docker
    - study
---

* content
{:toc}

>
> 本文对Tony Bai博文[再谈Docker容器单机网络：利用iptables trace和ebtables log](http://tonybai.com/2017/11/06/explain-docker-single-host-network-using-iptables-trace-and-ebtables-log/)的内容整理，对其中一些不懂的知识点补充，并增加图片了解。
> 
> 文中图片和日志均出自该博文。
>




## 简介

本文利用iptables和etables的日志，分析单机容器网络在`Container to Container`、`Local Process to Container`和`Container to External`三种场景下的数据包行走路径。

简易的容器网络拓扑如图1.：

<br/>
**图1.**
![](/img/study/study-15-docker-single-host-network-analysis/1-docker-network-topology.png)

文章基于netfilter数据流图2.来做数据路径分析。

<br/>
**图2.**
![](/img/study/study-15-docker-single-host-network-analysis/2-packet flow-in-netfilter-and-general-networking.png)

文章中实验机的iptables规则列表如下（只展示较为重要的filter和nat表的规则，输出规则number，便于后续match分析时查看）：

```
# iptables -nL --line-numbers -v -t filter
Chain INPUT (policy ACCEPT 2558K packets, 178M bytes)
num   pkts bytes target     prot opt in     out     source               destination

Chain FORWARD (policy DROP 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination
1       10   840 DOCKER-USER  all  --  *      *       0.0.0.0/0            0.0.0.0/0
2       10   840 DOCKER-ISOLATION  all  --  *      *       0.0.0.0/0            0.0.0.0/0
3        7   588 ACCEPT     all  --  *      docker0  0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
4        3   252 DOCKER     all  --  *      docker0  0.0.0.0/0            0.0.0.0/0
5        0     0 ACCEPT     all  --  docker0 !docker0  0.0.0.0/0            0.0.0.0/0
6        3   252 ACCEPT     all  --  docker0 docker0  0.0.0.0/0            0.0.0.0/0

Chain OUTPUT (policy ACCEPT 2460K packets, 214M bytes)
num   pkts bytes target     prot opt in     out     source               destination

Chain DOCKER (1 references)
num   pkts bytes target     prot opt in     out     source               destination

Chain DOCKER-ISOLATION (1 references)
num   pkts bytes target     prot opt in     out     source               destination
1       10   840 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0

Chain DOCKER-USER (1 references)
num   pkts bytes target     prot opt in     out     source               destination
1       10   840 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0

# iptables -nL --line-numbers -v -t nat
Chain PREROUTING (policy ACCEPT 884 packets, 46522 bytes)
num   pkts bytes target     prot opt in     out     source               destination
1      881 46270 DOCKER     all  --  *      *       0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL

Chain INPUT (policy ACCEPT 881 packets, 46270 bytes)
num   pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 1048K packets, 63M bytes)
num   pkts bytes target     prot opt in     out     source               destination
1        0     0 DOCKER     all  --  *      *       0.0.0.0/0           !127.0.0.0/8          ADDRTYPE match dst-type LOCAL

Chain POSTROUTING (policy ACCEPT 1048K packets, 63M bytes)
num   pkts bytes target     prot opt in     out     source               destination
1        0     0 MASQUERADE  all  --  *      !docker0  192.168.0.0/20       0.0.0.0/0

Chain DOCKER (2 references)
num   pkts bytes target     prot opt in     out     source               destination
1        0     0 RETURN     all  --  docker0 *       0.0.0.0/0            0.0.0.0/0
```

根据[参考2]和[参考3]对iptables中的一些细节进行分析：

* 1.对于filter表自定义链[参考2]`DOCKER-USER`、`DOCKER-ISOLATION`和`DOCKER`，会在filter表FORWARD链的规则1规则2和规则4match之后，跳转到这些自定义链中进入处理，再仔细看`DOCKER-USER`和`DOCKER-ISOLATION`的target都为`RETURN`，表示从该子链中跳回到触发该子链的父链规则中，从触发的父链规则继续匹配下一条规则[参考4]。
* 2.



## 参考

* *1.[CONTROLLING NETWORK RESOURCES USING CONTROL GROUPS](http://vger.kernel.org/netconf2009_slides/Network%20Control%20Group%20Whitepaper.odt)*
* *2.[iptables详解(10):iptables自定义链](http://www.zsythink.net/archives/1625)*
* *3.[Docker 网络学习笔记](http://lib.csdn.net/article/docker/1004)*
* *4.[iptables中的return](http://blog.csdn.net/wsclinux/article/details/53256494)*

