---
layout:     post
title:      "分层令牌桶原理---Hierachical token bucket theory「译」"
date:       2017-11-16 21:00:00 
author:     "guanjunjian"
categories: 网络基础知识
tags:
    - network
    - tc
    - study
---

* content
{:toc}

>
> 对[<<Hierachical token bucket theory>>](http://luxik.cdi.cz/~devik/qos/htb/manual/theory.htm)的翻译。
>




## 1. 定义

让我们来定义HTB的目标正式。首先是一些定义：

* Class：与Class有关的有，假设速率（assured rate AR），最高速率(ceil rate CR),优先级P，level和quantum Q。Class可以有父节点。还有变量R(actual rate)，代表数据包流离开该Class的速率，R每隔一小段时间就会测量一下。对于inner classes，它们的R等于所有子孙节点的R的和。
* Leaf:是那些子孙节点的class，只有leaf可以拥有数据包队列(packet queue)。
* Level：class的level决定了该class在分层（hierarchy）中的位置，叶子（Leaves）为level 0，根类(root classes)为LEVEL_COUNT-1，而inner class的level都比其父节点少一。可以看下图（LEVEL_COUNT=3 there）。
* Mode：class的mode是人为规定的值，由R、AR、和CR计算而来，可能的模式有：
*       Red: R > CR
*       Yellow: R <= CR and R > AR
*       Green otherwise
* D(c)：会列出所有想要从c这个父节点borrow的c的叶子节点。

## 2. 链路共享的目标

由R，我们可以定义**链路共享**的目标。对于每个class c，它需要做到

```
Rc = min(CRc, ARc + Bc)        [eq1]
```

其中Bc表示从祖先节点（ancestors）借来的速率，它的定义如下

```
       Qc Rp
Bc = -----------------------------  iff min[Pi over D(p)]>=Pc  [eq2]
     sum[Qi over D(p) where Pi=Pc]

Bc = 0   otherwise      [eq3]
```






## 参考

* *[Hierachical token bucket theory](http://luxik.cdi.cz/~devik/qos/htb/manual/theory.htm)*

