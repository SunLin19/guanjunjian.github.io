---
layout:     post
title:      "study11.HTB源码分析2---HTB"
date:       2017-11-20 16:00:00 
author:     "guanjunjian"
categories: 网络流量控制
tags:
    - network
    - tc
    - htb
    - study
---

* content
{:toc}

>
> 整理，并转载至[Linux内核中流量控制](http://cxw06023273.iteye.com/blog/867318)系列文章。
>  
> 来源：http://yfydz.cublog.cn
> 
> 内核代码版本为2.6.19.2
>




## 1. HTB(Hierarchical token bucket, 递阶令牌桶)

HTB, 从名称看就是TBF的扩展, 所不同的是TBF就一个节点处理所有数据, 而HTB是基于分类的流控方法, 以前那些流控一般用一个"tc qdisc"命令就可以完成配置, 而要配置好HTB, 通常情况下tc qdisc, class, filter三种命令都要用到, 用于将不同数据包分为不同的类别, 然后针对每种类别数据再设置相应的流控方法, 因此基于分类的流控方法远比以前所述的流控方法复杂。 

HTB将各种类别的流控处理节点组合成一个**节点树**, 每个叶节点是一个流控结构, 可在叶子节点使用不同的流控方法，如将pfifo, tbf等。HTB一个重要的特点是能设置每种类型的基本带宽，当本类带宽满而其他类型带宽空闲时可以向其他类型借带宽。注意这个树是**静态**的, 一旦TC命令配置好后就不变了, 而具体的实现是HASH表实现的, 只是逻辑上是树, 而且不是二叉树, 每个节点可以有多个子节点。

HTB**运行过程中**会将不同类别不同优先权的数据包进行有序排列，用到了有序表, 其数据结构实际是一种特殊的**二叉树**, 称为红黑树(Red Black Tree), 这种树的结构是**动态变化**的，而且数量不只一个，最大可有8×8个树。

红黑树的特征是:  

* 1) 每个节点不是红的就是黑的;  
* 2) 根节点必须是黑的;  
* 3) 所有叶子节点必须也是黑的;  
* 4) 每个红节点的子节点必须是黑的, 也就是红节点的父节点必须是黑节点;  
* 5) 从每个节点到最底层节点的所有路径必须包含相同数量的黑节点;  
  
关于红黑树的数据结构和操作在include/linux/rbtree.h和lib/rbtree.c中定义.

## 2. HTB操作结构定义

* `htb_cmode`：HTB操作数据包模式。HTB_CAN_SEND，可以发送, 没有阻塞；HTB_CANT_SEND，阻塞，不能发生数据包；HTB_MAY_BORROW，阻塞，可以向其他类借带宽来发送
* `htb_class`:HTB类别, 用于定义HTB的节点,包括`htb_class_leaf(叶子节点)`,`htb_class_inner(非叶子节点)`
* `htb_sched`:HTB私有数据结构
* `htb_class_ops`:HTB类别操作结构，对应于Qdisc_class_ops。有htb_graft(减子节点)、htb_leaf(增加子节点)、htb_walk(遍历)等
* `htb_qdisc_ops`:HTB流控操作结构

## 3. HTB的TC操作命令 

为了更好理解HTB各处理函数，先用HTB的配置实例过程来说明各种操作调用了哪些HTB处理函数，以下的配置实例取自HTB Manual, 属于最简单分类配置:

**3.1 配置网卡的根流控节点为HTB**

```
#根节点ID是0x10000, 缺省类别是0x10012,  
# handle x:y, x定义的是类别ID的高16位, y定义低16位  
#注意命令中的ID参数都被理解为16进制的数
tc qdisc add dev eth0 root handle 1: htb default 12
```


## 参考

* *[ux内核中流量控制(1)](http://cxw06023273.iteye.com/blog/867318)*

