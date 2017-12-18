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




## cls_cgroup_classify

`cls_cgroup_classify`

从该函数进入，主要想了解classid获取的过程，[代码](https://github.com/torvalds/linux/blob/v4.14-rc5/net/sched/cls_cgroup.c#L29#L44)如下：

```c
static int cls_cgroup_classify(struct sk_buff *skb, const struct tcf_proto *tp,
			       struct tcf_result *res)
{
	struct cls_cgroup_head *head = rcu_dereference_bh(tp->root);
	u32 classid = task_get_classid(skb);

	if (!classid)
		return -1;
	if (!tcf_em_tree_match(skb, &head->ematches, NULL))
		return -1;

	res->classid = classid;
	res->class = 0;

	return tcf_exts_exec(skb, &head->exts, res);
}
```

下面进入task_get_classid(skb)

## task_get_classid

`cls_cgroup_classify--->task_get_classid`

这里要分析的是`#ifdef CONFIG_CGROUP_NET_CLASSID`中的`task_get_classid()`

```c
static inline u32 task_get_classid(const struct sk_buff *skb)
{
	u32 classid = task_cls_state(current)->classid;

	/* Due to the nature of the classifier it is required to ignore all
	 * packets originating from softirq context as accessing `current'
	 * would lead to false results.
	 *
	 * This test assumes that all callers of dev_queue_xmit() explicitly
	 * disable bh. Knowing this, it is possible to detect softirq based
	 * calls by looking at the number of nested bh disable calls because
	 * softirqs always disables bh.
	 */
	if (in_serving_softirq()) {
		struct sock *sk = skb_to_full_sk(skb);

		/* If there is an sock_cgroup_classid we'll use that. */
		if (!sk || !sk_fullsock(sk))
			return 0;

		classid = sock_cgroup_classid(&sk->sk_cgrp_data);
	}

	return classid;
}
```

由于[net_next]中是在`n_serving_softirq()`中恢复了classid，所以这里主要分析`sock_cgroup_classid()`。

## sock_cgroup_classid

`cls_cgroup_classify--->task_get_classid--->sock_cgroup_classid`

```c
static inline u32 sock_cgroup_classid(struct sock_cgroup_data *skcd)
{
	/* fallback to 0 which is the unconfigured default classid */
	return (skcd->is_data & 1) ? skcd->classid : 0;
}
```

可以看到4.14.5的classid已经存储在了`sock_cgroup_data`这个结构体里，现在我们来看看这个结构体：

```c
struct sock_cgroup_data {
	union {
#ifdef __LITTLE_ENDIAN
		struct {
			u8	is_data;
			u8	padding;
			u16	prioidx;
			u32	classid;
		} __packed;
#else
		struct {
			u32	classid;
			u16	prioidx;
			u8	padding;
			u8	is_data;
		} __packed;
#endif
		u64		val;
	};
};
```

我们来看看注释中对该结构体的解释：

```
/*
 * sock_cgroup_data is embedded at sock->sk_cgrp_data and contains
 * per-socket cgroup information except for memcg association.
 *
 * On legacy hierarchies, net_prio and net_cls controllers directly set
 * attributes on each sock which can then be tested by the network layer.
 * On the default hierarchy, each sock is associated with the cgroup it was
 * created in and the networking layer can match the cgroup directly.
 *
 * To avoid carrying all three cgroup related fields separately in sock,
 * sock_cgroup_data overloads (prioidx, classid) and the cgroup pointer.
 * On boot, sock_cgroup_data records the cgroup that the sock was created
 * in so that cgroup2 matches can be made; however, once either net_prio or
 * net_cls starts being used, the area is overriden to carry prioidx and/or
 * classid.  The two modes are distinguished by whether the lowest bit is
 * set.  Clear bit indicates cgroup pointer while set bit prioidx and
 * classid.
 *
 * While userland may start using net_prio or net_cls at any time, once
 * either is used, cgroup2 matching no longer works.  There is no reason to
 * mix the two and this is in line with how legacy and v2 compatibility is
 * handled.  On mode switch, cgroup references which are already being
 * pointed to by socks may be leaked.  While this can be remedied by adding
 * synchronization around sock_cgroup_data, given that the number of leaked
 * cgroups is bound and highly unlikely to be high, this seems to be the
 * better trade-off.
 */
```

注释的主要内容是：

* 1.sock结构体的sk_cgrp_data就是sock_cgroup_data结构体，该结构体包含了每个socket的cgroup信息。
* 2.在legacy hierarchy(传统的层级，指的是cgroup v1)中，net_prio和net_cls可以设置sock的sk_cgrp_data，该信息可以被网络层检测到；在default hierarchy(指的是cgroup v2)中，sock与它所在的cgroup相关联，那么网络层就可以直接与cgroup匹配。
* 3.为了避免数据冗余，sock_cgroup_data对prioidx和classid进行了重载。在系统初启动时，sock_cgroup_data记录的是创建sock的cgroup，因此cgroup2可以直接与cgroup匹配([参考2]中提出,在cgroup2中，xt_cgroup可以直接与cgroup路径匹配)；一旦net_prio或net_cls启用了，那么prioidx和/或classid就代表的是它们字面的意思。可以根据sock_cgroup_data实例的最低位(lowest bit)来区分这两种模式，如果最低位是0，那么代表的是crgoup指针，如果是1，代表的是priodix和classid。
* 4.用户可以在任何时刻启用net_prio和net_cls，一旦他们被启用了，cgroup2匹配(xt_cgroup)将不在工作。当模式转换时，sock指向的cgroup引用有可能会溢出，而这通过同步添加sock_cgroup_data来弥补，但是这是在假设溢出的cgroup的数量有限且不多的情况下才成立的，这似乎是一个更好的权衡。




## 参考

* *1.[net_next](https://lists.linuxfoundation.org/pipermail/containers/2014-January/033844.html)*
* *2.[Rami Rosen： 理解CGroup v2](http://blog.csdn.net/juS3Ve/article/details/78769197)*
