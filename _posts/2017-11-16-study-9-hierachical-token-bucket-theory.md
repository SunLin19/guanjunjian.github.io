---
layout:     post
title:      "分层令牌桶原理---Hierachical token bucket theory「译」"
date:       2017-11-16 21:00:00 
author:     "guanjunjian"
categories: 网络基础知识
tags:
    - network
    - tc
    - htb
    - study
---

* content
{:toc}

>
> 对[<<Hierachical token bucket theory>>](http://luxik.cdi.cz/~devik/qos/htb/manual/theory.htm)的翻译。
>




## 1. 定义

让我们来定义HTB的目标正式。首先是一些定义：

* Class：与Class有关的有，假设速率（assured rate AR），最高速率(ceil rate CR),优先级P，level和quantum Q。Class可以有父节点。还有实际速率R(actual rate)，代表数据包流离开该Class的速率，R每隔一小段时间就会测量一下。对于inner classes，它们的R等于所有子孙节点的R的和。
* Leaf:是那些子孙节点的class，只有leaf可以拥有数据包队列(packet queue)。
* Level：class的level决定了该class在分层（hierarchy）中的位置，叶子（Leaves）为level 0，根类(root classes)为LEVEL_COUNT-1，而inner class的level都比其父节点少一。可以看下图（LEVEL_COUNT=3 there）。
* Mode：class的mode是人为规定的值，由R、AR、和CR计算而来，可能的模式有：
```
       Red: R > CR
       Yellow: R <= CR and R > AR
       Green otherwise
```
* D(c)：将会列出所有backlogged leaves，这些backlogged leaves是c的子孙节点，而这些backlogged leaves的class和c都是yellow状态。换言之，会列出所有想要从c这个父节点borrow的c的叶子节点。

## 2. 链路共享的目标

由R，我们可以定义**链路共享**的目标。对于每个class c，它需要做到

```
Rc = min(CRc, ARc + Bc)        [eq1]
```

其中Bc表示从祖先节点（ancestors）借来的速率，它的定义如下

```
/**
 * 对于[eq2]的解释
 * iff min[Pi over D(p)]>=Pc ： 当且仅当，p的所有需要借资源的子节点的优先级数值的最小值大于等于c的优先级数值，即在p中所有需要借资源的子节点中，c的优先级最高（包括一些节点优先级与c相等）。
 * 在以上情况下，Bc取值为，c的quantum 乘以 p的剩余速率的积，除以与c有相同优先级p叶子节点的个数（包括c）。
 */

/**
 * 对于[eq3]的解释
 * 当c的优先级不是D(p)中最高的时候，或者没有p节点，Bc=0
 */ 

             Qc Rp
Bc = -----------------------------  iff min[Pi over D(p)]>=Pc  [eq2]
     sum[Qi over D(p) where Pi=Pc]

Bc = 0   otherwise      [eq3]
```

其中`p`是`c`的父class。如果c没有父类，则Bc=0。对于Bc定义的`[eq2]`和`[eq3]`反应了优先级队列---当有优先级数值低于D(p)（优先级数值越小，优先级越高）的backlogged descendants（即要向p借资源的节点），那这些节点将被服务,而不是我们（these should be served, not us。怎么理解？）。上面的分式告诉我们，p的剩余速率（excess rate (Rp)）乘以Q，除以所有优先级与c相等的子节点的叶子类的个数，得到Bc。而p的剩余速率又由`[eq1]`和`[eq2]`共同定义，所以这是一个递归的过程。
说人话就是，当有class有需求且class的CR没有达到时，这些class的AR需要保持。剩余带宽应该被所有相同的拥有最高优先级的叶子节点平分，而这个平分需要根据Q值来计算。

## 3. 辅助延迟目标

我们同样需要确保class的隔离性。所以一个class的速率改变，不应该影响到别的class的延迟，除非被影响的class正好向同一个祖先节点借了资源。而且，当class从同level祖先节点借资源时，高优先级的class的延期应该低于低优先级的class。

## 4. CBQ提示

从[eq1]中可以看到，HTB的目标是CBQ目标子集的更严格定义。所以如果我们满足了HTB的目标，那也肯定满足了CBQ的目标。因此HTB是CBQ的一种。

## 5. HTB调度

在这里，我经常会把Linux实现的特定函数或变量名称放在parens中，这将帮助你阅读源码。

在HTB调度（struct htb_sched）中有一个class树(struct htb_class)。同时还有一个全局列表变量**self feed** list（htb_sched::row），它位于下图的最右边一列，self feed由self slot组成。每一个priority每一个level都有一个slot（one slot per priority per level）。因此在例子中有6个self slots（现在先忽略while slot，ignore while slot now）。每一个self slot拥有一个class列表(list)---list与class之间由彩色线条描述出对应关系。在同一个slot中的class拥有相同的level和相同的优先级。

self slot包含所有需求的green class列表（由D(c)设定）。

每个inner class（非叶子class）都有**inner feed** slot（htb_class::inner.feed）。每一个priority（红-优先级高，蓝-优先级低）每一个inner class都有一个inner feed slot。与self slot一样，inner feed slot也有一个class列表（list），这些class有同样的优先级，而且这些class必须是slot的拥有者inner class的孩子节点。

inner slot包含yellow children（由D(c)设定）。

![](/img/study/study-9-hierachical-token-bucket-theory/feed1.gif)
![](/img/study/study-9-hierachical-token-bucket-theory/feed2.gif)
 



## 参考

* *[Hierachical token bucket theory](http://luxik.cdi.cz/~devik/qos/htb/manual/theory.htm)*

