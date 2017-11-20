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
* `htb_class_ops`:HTB类别操作结构，对应于Qdisc_class_ops。有htb_graft(设置叶子节点的流控方法)、htb_leaf(增加子节点)、htb_walk(遍历)等
* `htb_qdisc_ops`:HTB流控操作结构

## 3. HTB的TC操作命令 

为了更好理解HTB各处理函数，先用HTB的配置实例过程来说明各种操作调用了哪些HTB处理函数，以下的配置实例取自HTB Manual, 属于最简单分类配置:

**3.1 配置网卡的根流控节点为HTB**

```
#根节点ID是0x10000, 缺省类别是0x10012,  
#handle x:y, x定义的是类别ID的高16位, y定义低16位  
#注意命令中的ID参数都被理解为16进制的数
tc qdisc add dev eth0 root handle 1: htb default 12
```

在内核中将调用`htb_init()`函数初始化HTB流控结构。

**3.2 建立分类树**

```
#根节点总流量带宽100kbps, 内部类别ID是0x10001  
tc class add dev eth0 parent 1: classid 1:1 htb rate 100kbps ceil 100kbps  
#第一类别数据分30kbps, 最大可用100kbps, 内部类别ID是0x10010(注意这里确实是16进制的10)  
tc class add dev eth0 parent 1:1 classid 1:10 htb rate 30kbps ceil 100kbps  
#第二类别数据分30kbps, 最大可用100kbps, 内部类别ID是0x10011  
tc class add dev eth0 parent 1:1 classid 1:11 htb rate 10kbps ceil 100kbps  
#第三类别(缺省类别)数据分60kbps, 最大可用100kbps, 内部类别ID是0x10012  
tc class add dev eth0 parent 1:1 classid 1:12 htb rate 60kbps ceil 100kbps
```

在内核中将调用htb_change_class()函数来修改HTB参数 

**3.3 数据包分类**

```
#对源地址为1.2.3.4, 目的端口是80的数据包为第一类, 0x10010  
tc filter add dev eth0 protocol ip parent 1:0 prio 1 u32 match ip src 1.2.3.4 match ip  
dport 80 0xffff flowid 1:10  
#对源地址是1.2.3.4的其他类型数据包是第2类, 0x10011  
tc filter add dev eth0 protocol ip parent 1:0 prio 1 u32 match ip src 1.2.3.4 flowid  
1:11  
#其他数据包将作为缺省类, 0x10012
```

在内核中将调用htb_find_tcf(), htb_bind_filter()函数来将为HTB绑定过滤表

**3.4 设置每个叶子节点的流控方法**

```
# 1:10节点为pfifo  
tc qdisc add dev eth0 parent 1:10 handle 20: pfifo limit 10  
# 1:11节点也为pfifo  
tc qdisc add dev eth0 parent 1:11 handle 30: pfifo limit 10  
# 1:12节点使用sfq, 扰动时间10秒  
tc qdisc add dev eth0 parent 1:12 handle 40: sfq perturb 10
```

在内核中会使用htb_leaf()查找HTB叶子节点, 使用htb_graft()函数来设置叶子节点的流控方法。

## 4. HTB类别操作

* `htb_graft`:嫁接,设置HTB叶子节点的流控方法
* `htb_leaf`:获取叶子节点
* `htb_get`:根据classid获取类，并增加引用计数
* `htb_put`:减少类别引用计数，如果减到0，释放该类别
* `htb_destroy_class`:释放类别,由于使用了递归处理, 因此HTB树不能太大, 否则就会使内核堆栈溢出而导致内核崩溃, HTB定  
义的最大深度是8层
* `htb_change_class`:更改类别结构内部参数,如普通速率和峰值速率 
* `htb_find_tcf`:查找过滤规则表
* `htb_bind_filter`:绑定过滤器
* `htb_unbind_filter`:解开过滤器
* `htb_walk`:遍历HTB
* `htb_dump_class`:类别参数输出,如速率、优先权值、定额(quantum)、层次值
* `htb_dump_class_stats`:类别统计信息输出 

## 5. HTB一些操作函数

**5.1 入队**

```c
static int htb_enqueue(struct sk_buff *skb, struct Qdisc *sch)  
{
	int ret;
	// HTB私有数据结构  
	struct htb_sched *q = qdisc_priv(sch);
	// 对数据包进行分类
	struct htb_class *cl = htb_classify(skb, sch, &ret);
	if (cl == HTB_DIRECT) {
	// 分类结果是直接发送
		/* enqueue to helper queue */
		// 如果直接发送队列中的数据包长度小于队列限制值, 将数据包添加到队列末尾
		if (q->direct_queue.qlen < q->direct_qlen) {
			__skb_queue_tail(&q->direct_queue, skb);
			q->direct_pkts++;
		} else {
			// 否则丢弃数据包
			kfree_skb(skb);
			sch->qstats.drops++;
			return NET_XMIT_DROP;
		}
#ifdef CONFIG_NET_CLS_ACT
// 定义了NET_CLS_ACT的情况(支持分类动作)
	} else if (!cl) {
		// 分类没有结果, 丢包
		if (ret == NET_XMIT_BYPASS)
			sch->qstats.drops++;
		kfree_skb(skb);
		return ret;
#endif
	// 有分类结果, 进行分类相关的叶子节点流控结构的入队操作
	} else if (cl->un.leaf.q->enqueue(skb, cl->un.leaf.q) !=
		   NET_XMIT_SUCCESS) {
		// 入队不成功的话丢包
		sch->qstats.drops++;
		cl->qstats.drops++;
		return NET_XMIT_DROP;
	} else {
		// 入队成功, 分类结构的包数字节数的统计数增加
		cl->bstats.packets++;
		cl->bstats.bytes += skb->len;
		// 激活HTB类别, 建立该类别的数据提供树, 这样dequeue时可以从中取数据包
		// 只有类别节点的模式是可发送和可租借的情况下才会激活, 如果节点是阻塞模式, 则不会被激活
		htb_activate(q, cl);
	}
	// HTB流控结构统计数更新, 入队成功
	sch->q.qlen++;
	sch->bstats.packets++;
	sch->bstats.bytes += skb->len;
	return NET_XMIT_SUCCESS;
}
```

大部分情况下数据包都不会进入直接处理队列, 而是进入各类别叶子节点, 因此入队的成功与否就在于叶子节点使用何种流控算法, 大都应该可以入队成功的， 入队不涉及类别节点模式的调整。 

**5.2 出队**

```c
```

**5.3 其他操作（不详细介绍）**

* `htb_requeue`:重入队
* `htb_dequeue_tree`:从指定的层次和优先权的RB树节点中取数据包
* `htb_lookup_leaf`:查找叶子分类节点
* `htb_charge_class`:关于类别节点的令牌和缓冲区数据的更新计算, 调整类别节点的模式
* `htb_change_class_mode`:调整类别节点的发送模式 
* `htb_class_mode`:计算类别节点的模式
* `htb_add_to_wait_tree`:将类别节点添加到等待树
* `htb_do_events(struct htb_sched *q, int level)`:对第level号等待树的类别节点进行模式调整
* `htb_delay_by`:HTB延迟处理 

HTB的流控就体现在通过令牌变化情况计算类别节点的模式, 如果是CAN_SEND就能继续发送数据, 该节点可以留在数据包提供树中; 如果阻塞CANT_SEND该类别节点就会从数据包提供树中拆除而不能发包; 如果模式是CAN_BORROW的则根据其他节点的带宽情况来确定是否能留在数据包提供树而继续发包, 不在提供树中, 即使有数据包也只能阻塞着不能发送, 这样就实现了流控管理。

为理解分类型流控算法的各种参数的含义, 最好去看一下RFC 3290。




## 参考

* *[ux内核中流量控制(11)](http://cxw06023273.iteye.com/blog/867337)*
* *[ux内核中流量控制(12)](http://cxw06023273.iteye.com/blog/867338)*
* *[ux内核中流量控制(13)](http://cxw06023273.iteye.com/blog/867339)*

