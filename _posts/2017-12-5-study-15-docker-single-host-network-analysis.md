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
> Tony Bai博文[再谈Docker容器单机网络：利用iptables trace和ebtables log](http://tonybai.com/2017/11/06/explain-docker-single-host-network-using-iptables-trace-and-ebtables-log/)读后的知识整理。
> 
> 文中图片和日志均出自该博文。
>




## 简介

本文利用iptables和etables的日志，分析单机容器网络在`Container to Container`、`Local Process to Container`和`Container to External`三种场景下的数据包行走路径。

简易的容器网络拓扑如图1.：

图1.
![](/img/study/study-15-docker-single-host-network-analysis/1-docker-network-topology.png)

文章基于netfilter数据流图2.来做数据路径分析。

图2.
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


## 参考

* *1.[CONTROLLING NETWORK RESOURCES USING CONTROL GROUPS](http://vger.kernel.org/netconf2009_slides/Network%20Control%20Group%20Whitepaper.odt)*

