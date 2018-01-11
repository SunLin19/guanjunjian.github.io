---
layout:     post
title:      "study17.veth_xmit分析"
date:       2018-01-03 10:00:00
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
> 目的是了解veth的数据包转发流程，了解classid是在何处被丢弃的
>




## 1. 使用方法

```shell
//使用veth  
//1.创建两块虚拟网卡veth1、veth2，然后点对点连接，此后两块网卡的数据会互相发送到对方  
$ ip link add veth1 type veth peer name veth2  
  
//2.创建网络命名空间t1  
$ ip netns add t1  
  
//3.将veth0加入t1，此时veth1便看不到了，因为被加入到其他命名空间中了  
$ ip link set veth1 netns t1  
  
//4.配置veth1的ip地址  
$ ip netns exec t1 ifconfig veth1 192.168.1.200/24  
  
//5.设置t1网络的默认路由  
$ ip netns exec t1 route add default gw 192.168.1.1  
  
//6.此时将veth2加入本地网桥中，便可以实现veth1在t1中访问外部网络了，过程略，可参考docker中的网络配置步骤 
```

## 2. 源码分析

源码位于`drivers\net\veth.c`

### 2.1 向ip link注册服务

```c
static __init int veth_init(void)
{
	return rtnl_link_register(&veth_link_ops);
}
```

其中`veth_link_ops`是与ip link一一对应的

```c
static struct rtnl_link_ops veth_link_ops = {
	.kind		= DRV_NAME,
	.priv_size	= sizeof(struct veth_priv),
	.setup		= veth_setup,
	.validate	= veth_validate,
	.newlink	= veth_newlink,
	.dellink	= veth_dellink,
	.policy		= veth_policy,
	.maxtype	= VETH_INFO_MAX,
	.get_link_net	= veth_get_link_net,
};
```

### 2.2 新加入一条veth链接

```c
static int veth_newlink(struct net *src_net, struct net_device *dev,
			struct nlattr *tb[], struct nlattr *data[],
			struct netlink_ext_ack *extack)
{
	int err;
	struct net_device *peer;
	struct veth_priv *priv;
	char ifname[IFNAMSIZ];
	struct nlattr *peer_tb[IFLA_MAX + 1], **tbp;
	unsigned char name_assign_type;
	struct ifinfomsg *ifmp;
	struct net *net;

	//首先创建并注册peer
	...
	//创建peer
	peer = rtnl_create_link(net, ifname, name_assign_type,
				&veth_link_ops, tbp);
	...
 	//注册peer
	err = register_netdevice(peer);
    ...
	//互相将peer保存到private中，在xmit的时候使用
	priv = netdev_priv(dev);
	rcu_assign_pointer(priv->peer, peer);

	priv = netdev_priv(peer);
	rcu_assign_pointer(priv->peer, dev);
	return 0;
}
```

上面的代码中涉及到了`netdev_priv()`，返回的是net设备的private数据，现在来看看这个数据是什么，根据[参考2][2]。

![](http://img.my.csdn.net/uploads/201112/21/0_13244603892DxH.gif)

而根据`veth_newlink()`可以找到，veth的private数据就是`veth_priv`结构体。

```c
struct veth_priv {
	struct net_device __rcu	*peer;
	atomic64_t		dropped;
	unsigned		requested_headroom;
};
```

## 2.3 初始化并启动veth设备

```c
static void veth_setup(struct net_device *dev)
{
	//以太网设备的通用初始化 
	ether_setup(dev);

	dev->priv_flags &= ~IFF_TX_SKB_SHARING;
	dev->priv_flags |= IFF_LIVE_ADDR_CHANGE;
	dev->priv_flags |= IFF_NO_QUEUE;
	dev->priv_flags |= IFF_PHONY_HEADROOM;
    
	//veth的操作列表，其中包括veth的发送函数veth_xmit
	dev->netdev_ops = &veth_netdev_ops;
	dev->ethtool_ops = &veth_ethtool_ops;
	dev->features |= NETIF_F_LLTX;
	dev->features |= VETH_FEATURES;
	dev->vlan_features = dev->features &
			     ~(NETIF_F_HW_VLAN_CTAG_TX |
			       NETIF_F_HW_VLAN_STAG_TX |
			       NETIF_F_HW_VLAN_CTAG_RX |
			       NETIF_F_HW_VLAN_STAG_RX);
	dev->needs_free_netdev = true;
	dev->priv_destructor = veth_dev_free;
	dev->max_mtu = ETH_MAX_MTU;

	dev->hw_features = VETH_FEATURES;
	dev->hw_enc_features = VETH_FEATURES;
	dev->mpls_features = NETIF_F_HW_CSUM | NETIF_F_GSO_SOFTWARE;
}
```

## 2.4 通过veth发送skb

```c
static netdev_tx_t veth_xmit(struct sk_buff *skb, struct net_device *dev)
{
	//获取veth的私有数据
	struct veth_priv *priv = netdev_priv(dev);
	struct net_device *rcv;
	int length = skb->len;

	rcu_read_lock();
	rcv = rcu_dereference(priv->peer);
	if (unlikely(!rcv)) {
		kfree_skb(skb);
		goto drop;
	}
	//调用dev_forward_skb向该veth的peer发送数据包
	if (likely(dev_forward_skb(rcv, skb) == NET_RX_SUCCESS)) {
	    //如果发送成功，则更新veth的数据包统计
		struct pcpu_vstats *stats = this_cpu_ptr(dev->vstats);

		u64_stats_update_begin(&stats->syncp);
		stats->bytes += length;
		stats->packets++;
		u64_stats_update_end(&stats->syncp);
	} else {
drop:
		atomic64_inc(&priv->dropped);
	}
	rcu_read_unlock();
	return NETDEV_TX_OK;
}
```

以上代码涉及到了rcu机制，可以阅读[参考3][3]来了解

-	1.宽期限（rcu_read_lock()、rcu_read_unlock()、synchronize_rcu()）
-	2.订阅---发布机制（rcu_assign_pointer()、rcu_dereference()）
-	3.数据读取的完整性。

从代码中看到，`veth_xmit()`调用了`dev_forward_skb(rcv, skb)`将数据包发送给veth peer，这部分将在[study18. dev_forward_skb分析][4]中继续分析。

## 参考

* *1[网络子系统87_veth实现][1]*
* *2.[网络驱动移植之例解netdev_priv函数][2]*
* *3.[linux内核 RCU机制详解][3]*

[1]:http://blog.csdn.net/nerdx/article/details/38561933 "网络子系统87_veth实现" 
[2]:http://blog.csdn.net/npy_lp/article/details/7090541 "网络驱动移植之例解netdev_priv函数" 
[3]:http://blog.csdn.net/xabc3000/article/details/15335131 "linux内核 RCU机制详解" 
[4]:https://guanjunjian.github.io/2018/01/05/study-18-dev_forward_skb-source-analysis/ "study18. dev_forward_skb分析"