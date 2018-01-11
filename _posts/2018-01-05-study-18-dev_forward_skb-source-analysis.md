---
layout:     post
title:      "study18.dev_forward_skb分析"
date:       2018-01-05 10:00:00
author:     "guanjunjian"
categories: 网络源码分析
tags:
    - study
    - network
---

* content
{:toc}

>
> 基于内核4.14.5
>
> 本文是[study17. veth_xmit分析][10]的后续分析
>




## 1. 代码调用过程

发送端(veth):

```
|--->dev_forward_skb()
    |--->__dev_forward_skb()   
    |    |--->____dev_forward_skb()
    |    |    |--->if()    // 判断是否可转发 
    |    |    |--->skb_scrub_packet() //清除skb可能破坏命名空间独立性的信息
    |    |    |    |--->skb_orphan()
    |    |    |        |--->skb->destructor()  //调用skb的destructor
    |    |    |        |--->skb->sk     = NULL  //将skb的sk字段设为NULL，这里classid就被丢弃了
    |    |    |--->skb->priority = 0
    |    |--->if (likely(!ret))   // 如果____dev_forward_skb执行正确 
    |         |--->skb->protocol = eth_type_trans()
    |         |--->skb_postpull_rcsum()
    |--->netif_rx_internal()  //如果__dev_forward_skb执行正确
         |--->enqueue_to_backlog()    //将数据包添加到per-cpu的接收队列中
              |--->__skb_queue_tail()  //将skb保存到input_pkt_queue队列中
              |--->____napi_schedule() //唤醒软中断
                  |--->list_add_tail()  //将该设备添加到softnet_data的poll_list队列中
                  |--->__raise_softirq_irqoff()  //唤醒软中断，调用对应的设备(veth peer)来收包
```

接收端(veth peer)

```
|--->net_rx_action()
    |--->process_backlog()
        |--->__netif_receive_skb()
            |--->__netif_receive_skb_core()
```

这里我们分析的是发送端的处理过程。

## 2. 注释

```
/**
 * dev_forward_skb - loopback an skb to another netif
 *
 * @dev: destination network device
 * @skb: buffer to forward
 *
 * return values:
 *  NET_RX_SUCCESS  (no congestion)
 *  NET_RX_DROP     (packet was dropped, but freed)
 *
 * dev_forward_skb can be used for injecting an skb from the
 * start_xmit function of one device into the receive queue
 * of another device.
 *
 * The receiving device may be in another namespace, so
 * we have to clear all information in the skb that could
 * impact namespace isolation.
 */
```

注释翻译如下：

-   dev_forward_skb函数的作用是回环发送数据包到另一个网络接口
-   参数：dev表示目的网络设备，如果veth调用的话，就是veth的peer；skb表示要发送的buffer
-   返回值NET_RX_SUCCESS表示发送成功，没有拥塞；NET_RX_DROP表示skb被丢弃并释放
-   dev_forward_skb可以被用于在一个网络设备的start_xmit函数中（对于veth就是veth_xmit函数）将一个skb注入到另一个网络设备的接收队列中
-   接收设备有可能存在另一个命名空间中，所以我们必须清楚所有可能破坏命名空间独立性的信息

从注释的最后一条可以看出，classid有可能就是因为会破坏命名空间的独立性而被清除掉了。

在`3.`中详细分析代码实现细节。

## 3. dev_forward_skb()

```c
int dev_forward_skb(struct net_device *dev, struct sk_buff *skb)
{
    return __dev_forward_skb(dev, skb) ?: netif_rx_internal(skb);
}
```

* 若`__dev_forward_skb(dev, skb)`结果为非0，表示出错，直接返回`__dev_forward_skb(dev, skb)`的结果
* 若`__dev_forward_skb(dev, skb)`结果为为0，表示成功，继续调用`netif_rx_internal(skb)`，并返回`netif_rx_internal(skb)`的结果

-   `__dev_forward_skb()`的工作是：
    -   判断是否符合转发条件
    -   清空skb可能破坏命名空间独立性的信息
    -   初始化skb的prorotocal字段，设置skb的dev字段
    -   更新skb的校验和
-   `netif_rx_internal()`的工作是：
    -   选择合适的cpu   
    -   将skb添加到per-cpu的接收队列中
    -   唤醒软中断，调用对应设备(veth peer)来收包

分别在`3.1`和`3.2`分析`__dev_forward_skb()`和`netif_rx_internal()`。

下面首先继续看`__dev_forward_skb(dev, skb)`

### 3.1 __dev_forward_skb()

```
--->dev_forward_skb()
    --->__dev_forward_skb()   <<<<<
        --->____dev_forward_skb(dev, skb)
        --->if (likely(!ret))
```

```c
int __dev_forward_skb(struct net_device *dev, struct sk_buff *skb)
{
    int ret = ____dev_forward_skb(dev, skb);

    if (likely(!ret)) {
        skb->protocol = eth_type_trans(skb, dev);
        skb_postpull_rcsum(skb, eth_hdr(skb), ETH_HLEN);
    }

    return ret;
}
```

这里主要做了两件事：

-   1.调用`____dev_forward_skb(dev, skb)`，该函数的工作是：
    -   判断是否符合转发条件（a. 能否将skb frags buffers拷贝到内核空间；b. 转发接口是否开启；c. skb的长度是否符合要求）
    -   清空skb可能破坏命名空间独立性的信息（最主要的是调用了skb的destructor和将skb->sk设为NULL，这里直接将classid丢弃）
-   2. 若`____dev_forward_skb(dev, skb)`执行顺利，返回`ret==0`，继续执行if()中的语句，if()语句的工作是：
        *  `skb->protocol = eth_type_trans(skb, dev)`:初始化skb的protocaol字段，链路层的接收函数`netif_receive_skb`会根据该字段来确定把报文送给哪个协议模块进一步处理。同时设置skb的dev字段为veth peer[[参考5][5]] 
        *  `skb_postpull_rcsum()`:更新skb的校验和

如果上述过程没有问题，将`ret==0`返回。

下面详细看`____dev_forward_skb(dev, skb)`。

#### ____dev_forward_skb()

```
--->dev_forward_skb()
    --->__dev_forward_skb()   
        --->____dev_forward_skb(dev, skb)   <<<<<
```

```c
static __always_inline int ____dev_forward_skb(struct net_device *dev,
                           struct sk_buff *skb)
{
    if (skb_orphan_frags(skb, GFP_ATOMIC) ||
        unlikely(!is_skb_forwardable(dev, skb))) {
        atomic_long_inc(&dev->rx_dropped);
        kfree_skb(skb);
        return NET_RX_DROP;
    }

    skb_scrub_packet(skb, true);
    skb->priority = 0;
    return 0;
}
```

-   1.`skb_orphan_frags(skb, GFP_ATOMIC)`将用户空间的skb frags buffers拷贝到内核空间中，如果拷贝成功则返回0，否则是错误代码
-   2.is_skb_forwardable(dev, skb)：
    -   如果接口没启动的，返回false
    -   如果sbk的长度没有超长，返回true
    -   如果skb的长度超长了，但开启了gso，依然返回true
    -   以上都没有返回true的话，就返回false

综上，if()条件语句表示的意思是，如果`1.用户到内核空间的拷贝失败2.dev接口没有开启3.skb长度超长且没有开启gso`则会执行if语句里的代码，即丢包的相关处理，并返回`NET_RX_DROP`。

如果以上if没有执行，则继续后继代码。下面进一步分析`skb_scrub_packet()`。

#### skb_scrub_packet()

```
--->dev_forward_skb()
    --->__dev_forward_skb()   
        --->____dev_forward_skb(dev, skb)   
            --->skb_scrub_packet(skb, true);   <<<<<<
```

首先来看看注释，如下：

```
/**
 * skb_scrub_packet - scrub an skb
 *
 * @skb: buffer to clean
 * @xnet: packet is crossing netns
 *
 * skb_scrub_packet can be used after encapsulating or decapsulting a packet
 * into/from a tunnel. Some information have to be cleared during these
 * operations.
 * skb_scrub_packet can also be used to clean a skb before injecting it in
 * another namespace (@xnet == true). We have to clear all information in the
 * skb that could impact namespace isolation.
 */
```

翻译如下：

```
/**
 * skb_scrub_packet - 作用是清空skb（终于找到了最终想要了解的东西）
 *
 * @skb:表示要清空的skb
 * @xnet:表示数据包是否跨网络命名空间，从上文看传入的是true
 *
 * skb_scrub_packet可以用于经过tunnel传输的数据包的封包或解包之后，在这些操作中，一些信息是必须清除的。
 * skb_scrub_packet也可以用来清除将要传输到另一个网络命名空间的数据包
 *（@xnet == true，即是我们目前分析的这种情况），我们必须清除所有可能破坏命名空间独立性的信息。
 */
```

从注释已经说明得很清楚了，在`____dev_forward_skb()`在调用`skb_scrub_packet(skb, true)`目的是清除skb的信息以避免破坏命名空间的独立性，那接下来我们详细看代码，看看清除了哪些信息。

```c
void skb_scrub_packet(struct sk_buff *skb, bool xnet)
{
    //记录时间戳
    skb->tstamp = 0;
    //表示数据包发给谁，PACKET_HOST表示发送给本机
    skb->pkt_type = PACKET_HOST;
    //数据包到达接口的索引号
    skb->skb_iif = 0;
    //赋值为0表示不允许本地分片（local fragmentation）
    skb->ignore_df = 0;
    //清除skb的destination entry，即路由项
    skb_dst_drop(skb);
    //与xfrm安全框架有关，清除skb的security path
    secpath_reset(skb);
    //一下两个函数调用都与netfilter packet trace flag，这里忽略详细分析
    nf_reset(skb);
    nf_reset_trace(skb);
    //如果不是跨网络命名空间传递数据包，那么到这里清除任务就结束了，但是从上文可以知道，传入的是true，所以还得继续
    if (!xnet)
        return;
    //将skb的ipvs_property设为0，该属性与ipvs（Linux Virtual Server）有关
    ipvs_reset(skb);
    skb_orphan(skb);
    //清除普通包标记
    skb->mark = 0;
}
```

以上还没有分析`skb_orphan(skb)`，下面进入到该函数中详细分析。

#### skb_orphan()

```
--->dev_forward_skb()
    --->__dev_forward_skb()   
        --->____dev_forward_skb(dev, skb)   
            --->skb_scrub_packet(skb, true);
                --->skb_orphan(skb)   <<<<<<
```

首先来看看注释，如下：

```
/**
 *  skb_orphan - orphan a buffer
 *  @skb: buffer to orphan
 *
 *  If a buffer currently has an owner then we call the owner's
 *  destructor function and make the @skb unowned. The buffer continues
 *  to exist but is no longer charged to its former owner.
 */
```

注释翻译如下：

```
/**
  * skb_orphan - 使一个buffer成为孤儿
  * @skb: 将要执行孤儿操作的skb
  *
  * 如果buffer现在还有拥有者，那么我们调用拥有者的destructor函数，让@skb成为无拥有者。
  * 执行该函数后，buffer忍让会存在，但是不由他之前的拥有者管理了
  */
```

下面来看代码：

```c
static inline void skb_orphan(struct sk_buff *skb)
{
    if (skb->destructor) {
        skb->destructor(skb);
        skb->destructor = NULL;
        skb->sk     = NULL;
    } else {
        BUG_ON(skb->sk);
    }
}
```

根据[[参考3][3]]：

> skb_orphan主要是回调了前一个属主赋予该skb的析构函数

可以看到`skb_orphan`主要完成了两个工作：

-   调用skb的`destructor`函数
-   将`skb->sk`置为NULL

根据[[参考4][4]]，可以知道classid是存储在`skb->sk->sk_cgrp_data->classid`，而这里skb->sk已经被清空，自然在veth转发后，classid就不存在了。

### 3.2 netif_rx_internal(skb)

根据[[参考9][9]]，`netif_rx_internal()`就是非NAPI设备对应的中断上半部。

这里说一下NAPI设备和非NAPI设备的区别：

-	NAPI设备：首次数据包的接收使用中断的方式，而后续的数据包就会使用轮询处理了，也就是`中断+轮询`，在设备调度处理数据期间，禁止中断
-	非NAPI设备：每次都是通过中断通知，在设备调度处理数据期间允许中断，会继续收包

若`3.1 __dev_forward_skb(dev, skb)`中的所有过程执行顺利，该函数返回0，`return __dev_forward_skb(dev, skb) ?: netif_rx_internal(skb);`将继续调用`netif_rx_internal()`，该小节将详细分析`netif_rx_internal()`的执行过程。

```c
static int netif_rx_internal(struct sk_buff *skb)
{
    int ret;
    //设置skb的tstamp值，此值是记录接收包的时间，@tstamp: Time we arrived/left
    net_timestamp_check(netdev_tstamp_prequeue, skb);
    
    trace_netif_rx(skb);
    
    if (static_key_false(&generic_xdp_needed)) {
        int ret;
        //禁止抢占
        preempt_disable();
        rcu_read_lock();
        ret = do_xdp_generic(rcu_dereference(skb->dev->xdp_prog), skb);
        rcu_read_unlock();
        preempt_enable();

        /* Consider XDP consuming the packet a success from
         * the netdev point of view we do not want to count
         * this as an error.
         */
        if (ret != XDP_PASS)
            return NET_RX_SUCCESS;
    }

//RPS 和 RFS 相关代码；RPS: Receive Packet Steering (接收端包的控制)；RFS: Receive Flow Steering (接收端流的控制)；对于多处理器系统，当使能了RPS（receive packet steering）功能，则会根据负载均衡的原则将数据包平均分配到各个CPU，减少单个CPU的工作负荷[参考7]
#ifdef CONFIG_RPS
    if (static_key_false(&rps_needed)) {
        struct rps_dev_flow voidflow, *rflow = &voidflow;
        int cpu;

        preempt_disable();
        rcu_read_lock();
        
        //选择合适的CPU id
        cpu = get_rps_cpu(skb->dev, skb, &rflow);
        if (cpu < 0)
            cpu = smp_processor_id();
        
        //将skb入队
        ret = enqueue_to_backlog(skb, cpu, &rflow->last_qtail);

        rcu_read_unlock();
        preempt_enable();
    } else
#endif
    {
        unsigned int qtail;
        
        //将skb入队
        ret = enqueue_to_backlog(skb, get_cpu(), &qtail);
        put_cpu();
    }
    return ret;
}
```

从以上代码可以看出，`netif_rx_internal()`的功能是获取适当的cpu，并将skb添加到cpu的per-CPU接收队列中。

每个per-cpu收包队列都是一个`softnet_data`结构体，该结构体大致如下：

```c
struct softnet_data {
    ...
    struct list_head    poll_list;  //连接所有的轮询设备
    struct sk_buff_head process_queue;  //用于非NAPI设备，因为NAPI设备有自己的队列，处理数据时从该队列取，负责处理
    struct sk_buff_head input_pkt_queue;  //用于非NAPI设备，因为NAPI设备有自己的队列，数据包到来时首先填充到该队列，负责接收
    struct napi_struct  backlog;  //用于NAPI设备，代表一个虚拟设备供轮询使用，当轮询到该设备时，就会使用以上两个队列
    ...
}
```

根据[[参考1][1]]可以知道：

> 内核中定义了全局的per-cpu收包队列softnet_data。struct softnet_data结构体中，poll_list是NAPI设备列表；input_pkt_queue为per-cpu的收包队列；backlog是默认的处理收包的napi设备。

> 大致关系如下: 每个cpu有一个softnet_data结构，发送到包都放在了input_pkt_queue里面，所有的NAPI device都在poll_list里面，默认处理的NAPI device在backlog指向。

如下图：

![][7]

> 简而言之, 让某个网卡NIC(1) 处理某个包 P(a)要做的事情,  
就是把这个网卡的net_device放到softnet_data的poll_list上,   然后将sk_buff (就是P(a))的net_device的指针指向这个网卡的net_device (NIC(1)). 

> 包放到CPU的softnet_data的input_pkt_queue上，然后调用软中断进行处理。软中断调用poll，poll对于veth则是默认的内核实现(process_backlog)。

下面分析再详细看看入队函数`enqueue_to_backlog()`

#### enqueue_to_backlog()

调用`enqueue_to_backlog`函数可以将一个skb添加到指定的per-cpu的backlog queue中

```c
static int enqueue_to_backlog(struct sk_buff *skb, int cpu,
                  unsigned int *qtail)
{
    struct softnet_data *sd;
    unsigned long flags;
    unsigned int qlen;
    //获得指定cpu的softnet_data结构体
    sd = &per_cpu(softnet_data, cpu);
    //关闭中断
    local_irq_save(flags);

    rps_lock(sd);
    //如果skb要到达的dev，在本文中分析的即veth peer是关闭的，则直接跳到drop标签，即丢包
    if (!netif_running(skb->dev))
        goto drop;
    //获取softnet_data中input_pkt_queue队列的长度
    qlen = skb_queue_len(&sd->input_pkt_queue);
    //如果input_pkt_queue队列的长度没有超过设备队列长度最大值，其中skb_flow_limit与上文提到的RPS有关
    if (qlen <= netdev_max_backlog && !skb_flow_limit(skb, qlen)) {
        //如果input_pkt_queue不为空，说明虚拟设备已经得到调度，此时仅仅把数据加入input_pkt_queue队列即可
        if (qlen) {
enqueue:
            //将skb保存到input_pkt_queue队列中
            __skb_queue_tail(&sd->input_pkt_queue, skb);
            input_queue_tail_incr_save(sd, qtail);
            rps_unlock(sd);
            local_irq_restore(flags);
            return NET_RX_SUCCESS;
        }

        /* Schedule NAPI for backlog device
         * We can use non atomic operation since we own the queue lock
         */
        //只有qlen是0(表示虚拟设备没有被调度)的时候才执行到这里
        /*
        否则需要调度backlog即虚拟设备，然后再入队。napi_struct实例backlog中的state字段如果标记了NAPI_STATE_SCHED,则表明该设备已经在调度，不需要再次调度
        NAPI_STATE_SCHED：表示设备将在内核的下一次循环时被轮询
        NAPI_STATE_DISABLE：表示轮询已经结束且没有更多的分组等待处理，但设备并没有从poll list移除
        */
        if (!__test_and_set_bit(NAPI_STATE_SCHED, &sd->backlog.state)) {
            //未处于调度状态
            if (!rps_ipi_queued(sd))
                /*
                该函数把设备对应的napi_struct结构（每个设备对应一个napi_struct结构）插入到softnet_data的poll_list链表尾部，然后唤醒软中断，这样在下次软中断得到处理时，中断下半部就会得到处理。
				对于NAPI设备，传入的是自己的napi_struct，而对于非NAPI设备，则传入虚拟设备backlog。
                */
                ____napi_schedule(sd, &sd->backlog);
        }
        goto enqueue;
    }

drop:
    sd->dropped++;
    rps_unlock(sd);

    local_irq_restore(flags);

    atomic_long_inc(&skb->dev->rx_dropped);
    kfree_skb(skb);
    return NET_RX_DROP;
}
```

接下来看看`____napi_schedule(sd, &sd->backlog)`是如何实现的。

#### ____napi_schedule()

```c
static inline void ____napi_schedule(struct softnet_data *sd,
                     struct napi_struct *napi)
{
    //将该设备添加到softnet_data的poll_list队列中
    list_add_tail(&napi->poll_list, &sd->poll_list);
    //唤醒软中断，调用默认或者对应的NAPI设备(veth peer)来收包
    __raise_softirq_irqoff(NET_RX_SOFTIRQ);
}
```

根据[[参考8][8]]从`netif_rx_internal()`到`____napi_schedule()`是中断处理的上半部，当调用了`__raise_softirq_irqoff()`则就进入中断处理的下半部。

无论是NAPI接口还是非NAPI最后都是使用 net_rx_action 作为软中断处理函数。

根据[[参考1][1]]可知：

veth的接收函数没有定义, 使用默认的poll函数（非NAPI）, 也就是`process_backlog`，非NAPI默认调用过程如下：

```
--->net_rx_action()
    --->process_backlog()
        --->__netif_receive_skb()
            --->__netif_receive_skb_core()
```

然后数据包进入网络层。

下面看看`process_backlog()`的实现。

#### process_backlog()

```c
static int process_backlog(struct napi_struct *napi, int quota)
{
	struct softnet_data *sd = container_of(napi, struct softnet_data, backlog);
	bool again = true;
	int work = 0;

	/* Check if we have pending ipi, its better to send them now,
	 * not waiting net_rx_action() end.
	 */
	if (sd_has_rps_ipi_waiting(sd)) {
		local_irq_disable();
		net_rps_action_and_irq_enable(sd);
	}

	napi->weight = dev_rx_weight;
	while (again) {
		struct sk_buff *skb;
		/*
			涉及到两个队列process_queue和input_pkt_queue，数据包到来时首先填充input_pkt_queue，
			而在处理时从process_queue中取，根据这个逻辑，首次处理process_queue必定为空，检查input_pkt_queue
			如果input_pkt_queue不为空，则把其中的数据包迁移到process_queue中，然后继续处理，减少锁冲突。
		*/
		while ((skb = __skb_dequeue(&sd->process_queue))) {
			rcu_read_lock();
			__netif_receive_skb(skb);
			rcu_read_unlock();
			input_queue_head_incr(sd);
			if (++work >= quota)
				return work;

		}

		local_irq_disable();
		rps_lock(sd);
		if (skb_queue_empty(&sd->input_pkt_queue)) {
			/*
			 * Inline a custom version of __napi_complete().
			 * only current cpu owns and manipulates this napi,
			 * and NAPI_STATE_SCHED is the only possible flag set
			 * on backlog.
			 * We can use a plain write instead of clear_bit(),
			 * and we dont need an smp_mb() memory barrier.
			 */
			napi->state = 0;
			again = false;
		} else {
			skb_queue_splice_tail_init(&sd->input_pkt_queue,
						   &sd->process_queue);
		}
		rps_unlock(sd);
		local_irq_enable();
	}

	return work;
}
```

根据[[参考9][9]]:

>需要注意的每次处理都携带一个配额，即本次只能处理quota个数据包，如果超额了，即使没处理完也要返回，这是为了保证处理器的公平使用。
>处理在一个while循环中完成，循环条件正是work < quota，首先会从process_queue中取出skb,调用__netif_receive_skb上传给协议栈，然后增加work。
>当work即将大于quota时，即++work >= quota时，就要返回。
>当work还有剩余额度，但是process_queue中数据处理完了，就需要检查input_pkt_queue，因为在具体处理期间是开中断的，那么期间就有可能有新的数据包到来。
>如果input_pkt_queue不为空，则调用skb_queue_splice_tail_init函数把数据包迁移到process_queue。
>如果剩余额度足够处理完这些数据包，那么就把虚拟设备移除轮询队列。

自此，`dev_forward_skb`的整个流程分析完毕。

## 参考

* *1.[关于veth的发送和接收packet][1]*
* *2.[veth在内核的实现][2]*
* *3.[Linux内核中网络数据包的接收-第一部分 概念和框架][3]*
* *4.[study16.[net_cls]cls_cgroup_classify()分析---classid的存储][4]*
* *5.[链路层和网络层的接口 （linux网络子系统学习 第五节 ）][5]*
* *6.[内核接收分组理解][6]*
* *7.[网络数据包接收之GRO处理][8]*
* *8.[Linux NAPI处理流程分析][9]*

---

[1]:https://focusvirtualization.blogspot.com/2016/06/protocol-stack-20-vethpacket.html "关于veth的发送和接收packet"
[2]:http://ju.outofmemory.cn/entry/187069 "veth在内核的实现"
[3]:http://blog.csdn.net/dog250/article/details/50528280 "Linux内核中网络数据包的接收-第一部分 概念和框架"
[4]:https://guanjunjian.github.io/2017/12/14/study-16-cgroup-cls_cgroup_classify/ "study16.[net_cls]cls_cgroup_classify()分析---classid的存储"
[5]:http://blog.51cto.com/yaoyang/1269713 "链路层和网络层的接口 （linux网络子系统学习 第五节 ）"
[6]:http://www.cnblogs.com/lxgeek/p/4182029.html "内核接收分组理解"
[7]:https://raw.githubusercontent.com/guanjunjian/guanjunjian.github.io/master/img/study/study-18-dev_forward_skb-source-analysis/softnet_data.png "softnet_data图片"
[8]:http://blog.csdn.net/u011955950/article/details/41445791 "网络数据包接收之GRO处理"
[9]:https://www.cnblogs.com/ck1020/p/6838234.html   "Linux NAPI处理流程分析"
[10]:https://guanjunjian.github.io/2018/01/03/study-17-veth_xmit-source-analysis/ "study17. veth_xmit分析"



