---
layout:     post
title:      "study8./proc/net/tcp分析"
date:       2017-11-9 11:00:00 
author:     "guanjunjian"
categories: 网络基础知识
tags:
    - network
    - study
---

* content
{:toc}

> /proc/net/tcp文件提供了tcp的连接信息，是由net/ipv4/tcp_ipv4.c中的tcp4_seq_show()实现信息打印的
>
>本文内容来源于[linux官方文档proc_net_tcp.txt](https://github.com/torvalds/linux/blob/v4.10/Documentation/networking/proc_net_tcp.txt)
>




## 1. 官方文档解释

proc_net_tcp.txt介绍了/proc/net/tcp和/proc/net/tcp6接口。这些接口展示了tcp的连接信息。

展示的信息中，首先是listening TCP sockets，接着是established TCP sockets。

在linux中执行cat /proc/net/tcp，输出结果如图1：

![](/img/study/study-8-proc-net-tcp-analysis/img-1-execute-result.png)

文档中对其中每个字段的解释如下（由于长度原因，分为3个部分）：

<br/>
**part1:**

```
   46: 010310AC:9C4C 030310AC:1770 01 
   |      |      |      |      |   |--> connection state（套接字状态）
   |      |      |      |      |------> remote TCP port number（远端端口，主机字节序）
   |      |      |      |-------------> remote IPv4 address（远端IP，网络字节序）
   |      |      |--------------------> local TCP port number（本地端口，主机字节序）
   |      |---------------------------> local IPv4 address（本地IP，网络字节序）
   |----------------------------------> number of entry
```

`connection state`(套接字状态)，不同的数值代表不同的状态，参照如下：

```
TCP_ESTABLISHED:1   TCP_SYN_SENT:2
TCP_SYN_RECV:3      TCP_FIN_WAIT1:4
TCP_FIN_WAIT2:5     TCP_TIME_WAIT:6
TCP_CLOSE:7         TCP_CLOSE_WAIT:8
TCP_LAST_ACL:9      TCP_LISTEN:10
TCP_CLOSING:11
```

<br/>
**part2:**

```
   00000150:00000000 01:00000019 00000000  
      |        |     |     |       |--> number of unrecovered RTO timeouts（超时重传次数）
      |        |     |     |----------> number of jiffies until timer expires（超时时间，单位是jiffies）
      |        |     |----------------> timer_active (定时器类型，see below)
      |        |----------------------> receive-queue（根据状态不同有不同表示,see below）
      |-------------------------------> transmit-queue(发送队列中数据长度)
```

`receive-queue`，当状态是ESTABLISHED，表示接收队列中数据长度；状态是LISTEN，表示已经完成连接队列的长度。

`timer_active`:

```
  0  no timer is pending  //没有启动定时器
  1  retransmit-timer is pending  //重传定时器
  2  another timer (e.g. delayed ack or keepalive) is pending  //连接定时器、FIN_WAIT_2定时器或TCP保活定时器
  3  this is a socket in TIME_WAIT state. Not all fields will contain data (or even exist)  //TIME_WAIT定时器
  4  zero window probe timer is pending  //持续定时器
```

<br/>
**part3:**

```
   1000        0 54165785 4 cd1e6040 25 4 27 3 -1
    |          |    |     |    |     |  | |  | |--> slow start size threshold, 
	                                                or -1 if the threshold is >=0xFFFF 
    |          |    |     |    |     |  | |  |     （如果慢启动阈值大于等于0xFFFF则显示-1，否则表示慢启动阈值）
    |          |    |     |    |     |  | |  |      
    |          |    |     |    |     |  | |  |----> sending congestion window（当前拥塞窗口大小）
    |          |    |     |    |     |  | |-------> (ack.quick<<1)|ack.pingpong
	                                               （快速确认数和是否启用的标志位的或运算结果）    
    |          |    |     |    |     |  |---------> Predicted tick of soft clock (delayed ACK control data)
	                                               （用来计算延时确认的估值）
    |          |    |     |    |     |             
    |          |    |     |    |     |------------> retransmit timeout（）（RTO，单位是clock_t）
    |          |    |     |    |------------------> location of socket in memory（socket实例的地址）
    |          |    |     |-----------------------> socket reference count（socket结构体的引用数）
    |          |    |-----------------------------> inode（套接字对应的inode）
    |          |----------------------------------> unanswered 0-window probes（see below）
    |---------------------------------------------> uid（用户id）
```

`unanswered 0-window probes`：持续定时器或保活定时器周期性发送出去但未被确认的TCP段数目，在收到ACK之后清零

## 2. tcp4_seq_show()

`tcp4_seq_show()`函数位于[net/ipv4/tcp_ipv4.c#L2306#L2329](https://github.com/torvalds/linux/blob/master/net/ipv4/tcp_ipv4.c#L2306#L2329)。

首先需要来认识一下`struct seq_file`，称为序列文件接口。在UNIX的世界里，文件是最普通的概念，所以用文件来作为内核和用户空间传递数据的接口也是再普通不过的事情，并且这样的接口对于shell也是相当友好的，方便管理员通过shell直接管理系统。由于伪文件系统proc文件系统在处理大数据结构（大于一页的数据）方面有比较大的局限性，使得在那种情况下进行编程特别别扭，很容易导致bug，所以序列文件接口被发明出来，它提供了更加友好的接口，以方便程序员。之所以选择序列文件接口这个名字，应该是因为它主要用来导出一条条的记录数据。
Seq_file的实现基于proc文件。使用Seq_file，用户必须抽象出一个链接对象，然后可以依次遍历这个链接对象。这个链接对象可以是链表，数组，哈希表等等。有两个重要的结构体：1.`seq_operations`：定义了start、next、show、stop四个操作函数；2.`seq_file`：该结构会在seq_open函数调用中分配，然后作为参数传递给每个seq_file的操作函数。Privat变量可以用来在各个操作函数之间传递参数

下面是代码：

```c
static int tcp4_seq_show(struct seq_file *seq, void *v)
{
	struct tcp_iter_state *st;
	struct sock *sk = v;
	//set padding width
	seq_setwidth(seq, TMPSZ - 1);
	//SEQ_START_TOKEN：通常这个值传递给show的时候，show会打印表格头
	if (v == SEQ_START_TOKEN) {
		//向表头输入到seq中
		seq_puts(seq, "  sl  local_address rem_address   st tx_queue "
			   "rx_queue tr tm->when retrnsmt   uid  timeout "
			   "inode");
		goto out;
	}
	//指向文件的私有数据，是特例化一个序列文件的方法
	st = seq->private;
	if (sk->sk_state == TCP_TIME_WAIT)
		get_timewait4_sock(v, seq, st->num);
	else if (sk->sk_state == TCP_NEW_SYN_RECV)
		get_openreq4(v, seq, st->num);
	else
		//下面分析
		get_tcp4_sock(v, seq, st->num);
out:
	seq_pad(seq, '\n');
	return 0;
}
```

接下来看`get_tcp4_sock()`，主要代码如下：

```c
static void get_tcp4_sock(struct sock *sk, struct seq_file *f, int i)
{
	int timer_active;
	unsigned long timer_expires;
	const struct tcp_sock *tp = tcp_sk(sk);
	const struct inet_connection_sock *icsk = inet_csk(sk);
	const struct inet_sock *inet = inet_sk(sk);
	const struct fastopen_queue *fastopenq = &icsk->icsk_accept_queue.fastopenq;
	__be32 dest = inet->inet_daddr;
	__be32 src = inet->inet_rcv_saddr;
	//将一个16位数由网络字节顺序转换为主机字节顺序
	__u16 destp = ntohs(inet->inet_dport);
	__u16 srcp = ntohs(inet->inet_sport);
	int rx_queue;
	int state;
	//获取定时器类型（timer_active）和超时时间（timer_expires）
	if (icsk->icsk_pending == ICSK_TIME_RETRANS ||
	    icsk->icsk_pending == ICSK_TIME_EARLY_RETRANS ||
	    icsk->icsk_pending == ICSK_TIME_LOSS_PROBE) {
		timer_active	= 1;
		timer_expires	= icsk->icsk_timeout;
	} else if (icsk->icsk_pending == ICSK_TIME_PROBE0) {
		timer_active	= 4;
		timer_expires	= icsk->icsk_timeout;
	} else if (timer_pending(&sk->sk_timer)) {
		timer_active	= 2;
		timer_expires	= sk->sk_timer.expires;
	} else {
		timer_active	= 0;
		timer_expires = jiffies;
	}
	//read sk->sk_state for lockless contexts
	state = sk_state_load(sk);
	//根据tcp的状态，获取receive-queue的值
	if (state == TCP_LISTEN)
		rx_queue = sk->sk_ack_backlog;
	else
		/* Because we don't lock the socket,
		 * we might find a transient negative value.
		 */
		rx_queue = max_t(int, tp->rcv_nxt - tp->copied_seq, 0);
	//将tcp的信息加入到f，即传入的seq_file中
	seq_printf(f, "%4d: %08X:%04X %08X:%04X %02X %08X:%08X %02X:%08lX "
			"%08X %5u %8d %lu %d %pK %lu %lu %u %u %d",
		i, src, srcp, dest, destp, state,
		tp->write_seq - tp->snd_una,
		rx_queue,
		timer_active,
		jiffies_delta_to_clock_t(timer_expires - jiffies),
		icsk->icsk_retransmits,
		from_kuid_munged(seq_user_ns(f), sock_i_uid(sk)),
		icsk->icsk_probes_out,
		sock_i_ino(sk),
		atomic_read(&sk->sk_refcnt), sk,
		jiffies_to_clock_t(icsk->icsk_rto),
		jiffies_to_clock_t(icsk->icsk_ack.ato),
		(icsk->icsk_ack.quick << 1) | icsk->icsk_ack.pingpong,
		tp->snd_cwnd,
		state == TCP_LISTEN ?
		    fastopenq->max_qlen :
		    (tcp_in_initial_slowstart(tp) ? -1 : tp->snd_ssthresh));
}
```

## 3. /proc/net/tcp的创建

**tcp4_proc_init_net()**

`/proc/net/tcp`的初始化函数为`tcp4_proc_init_net()`，位于[net/ipv4/tcp_ipv4.c#L2348#L2350](https://github.com/torvalds/linux/blob/master/net/ipv4/tcp_ipv4.c#L2306#L2329)。

```c
static int __net_init tcp4_proc_init_net(struct net *net)
{
	return tcp_proc_register(net, &tcp4_seq_afinfo);
}

```

<br/>
首先来看一下`tcp4_seq_afinfo`

```c
static struct tcp_seq_afinfo tcp4_seq_afinfo = {
	.name		= "tcp",
	.family		= AF_INET,
	.seq_fops	= &tcp_afinfo_seq_fops,
	.seq_ops	= {
		.show		= tcp4_seq_show,
	},
};
```

可以看到`name="tcp"`，也就是一会要创建的文件名字是tcp。`.seq_ops.show=tcp4_seq_show()`应该是`cat /proc/net/tcp`时会调用的函数。

<br/>
**tcp4_proc_init_net()-->tcp_proc_register()**

接下来看` tcp_proc_register()`

```c
int tcp_proc_register(struct net *net, struct tcp_seq_afinfo *afinfo)
{
	int rc = 0;
	//proc目录结构体，定义位于fs/proc/internal.h
	struct proc_dir_entry *p;
	//赋值afinfo，也就是tcp4_seq_afinfo的seq_operations
	afinfo->seq_ops.start		= tcp_seq_start;
	afinfo->seq_ops.next		= tcp_seq_next;
	afinfo->seq_ops.stop		= tcp_seq_stop;
	//下面接着分析
	p = proc_create_data(afinfo->name, S_IRUGO, net->proc_net,
			     afinfo->seq_fops, afinfo);
	if (!p)
		rc = -ENOMEM;
	return rc;
}
``` 

<br/>
**tcp4_proc_init_net()-->tcp_proc_register()-->proc_create_data()**

下面接着看`proc_create_data()`

```c
/*
 * @name: afinfo->name,即"tcp"，你要建立的文件名
 * @mode: 读写权限，是一个八进制，传入的是S_IRUGO，即(S_IRUSR|S_IRGRP|S_IROTH)=00444
 * @parent: 指的应该是"/pro/net/"目录
 * @proc_fops: 传入的是afinfo->seq_fops,所以应该是tcp_afinfo_seq_fops
 * @data: 传入的是afinfo，即tcp4_seq_afinfo
 */
struct proc_dir_entry *proc_create_data(const char *name, umode_t mode,
					struct proc_dir_entry *parent,
					const struct file_operations *proc_fops,
					void *data)
{
	struct proc_dir_entry *pde;
	//00444 & 00170000 == 0，所以mode==170444
	if ((mode & S_IFMT) == 0)
		mode |= S_IFREG;
	//S_ISREG(170444)==(((170444) & S_IFMT) == S_IFREG)==(((00170444) & 00170000) == 0100000)==(170000==100000)==false
	//？？
	if (!S_ISREG(mode)) {
		WARN_ON(1);	/* use proc_mkdir() */
		return NULL;
	}
	BUG_ON(proc_fops == NULL);
	//170444 & 7777 == 444,所以mode==170444
	if ((mode & S_IALLUGO) == 0)
		mode |= S_IRUGO;
	//获得一个初始化了的proc_dir_entry结构体，下面继续分析
	pde = __proc_create(&parent, name, mode, 1);
	if (!pde)
		goto out;
	//pde->proc_fops==tcp_afinfo_seq_fops
	pde->proc_fops = proc_fops;
	//pde->data==tcp4_seq_afinfo
	pde->data = data;
	pde->proc_iops = &proc_file_inode_operations;
	//下面接着分析proc_register()
	if (proc_register(parent, pde) < 0)
		goto out_free;
	return pde;
out_free:
	kfree(pde);
out:
	return NULL;
}
	
}
```

看完了`proc_create_data()`，下面再分别看`__proc_create()`和`proc_register()`

<br/>
**tcp4_proc_init_net()-->tcp_proc_register()-->proc_create_data()-->__proc_create()**

首先来看`__proc_create()`，分析如下：

```c
/*
 * @ parent:指的应该是"/pro/net/"目录
 * @ name: "tcp"
 * @ mode: 170444
 * @ nlink: 1
 */
static struct proc_dir_entry *__proc_create(struct proc_dir_entry **parent,
					  const char *name,
					  umode_t mode,
					  nlink_t nlink)
{
	struct proc_dir_entry *ent = NULL;
	const char *fn;
	/*
	 * "quick string" -- eases parameter passing, but more importantly
 	 * saves "metadata" about the string (ie length and the hash).
 	 * /
	struct qstr qstr;
	//fn=tcp
	if (xlate_proc_name(name, parent, &fn) != 0)
		goto out;
	qstr.name = fn;
	qstr.len = strlen(fn);
	//对文件名字的长度进行检验
	if (qstr.len == 0 || qstr.len >= 256) {
		WARN(1, "name len %u\n", qstr.len);
		return NULL;
	}
	if (*parent == &proc_root && name_to_int(&qstr) != ~0U) {
		WARN(1, "create '/proc/%s' by hand\n", qstr.name);
		return NULL;
	}
	if (is_empty_pde(*parent)) {
		WARN(1, "attempt to add to permanently empty directory");
		return NULL;
	}
	//分配内存空间,由于proc_dir_entry结构体的name为char name[]，所以这里要为proc_dir_entry加上名字的长度
	ent = kzalloc(sizeof(struct proc_dir_entry) + qstr.len + 1, GFP_KERNEL);
	if (!ent)
		goto out;
	//给ent的name赋值
	memcpy(ent->name, fn, qstr.len + 1);	
	ent->namelen = qstr.len;
	ent->mode = mode;
	ent->nlink = nlink;
	ent->subdir = RB_ROOT;
	atomic_set(&ent->count, 1);
	spin_lock_init(&ent->pde_unload_lock);
	INIT_LIST_HEAD(&ent->pde_openers);
	proc_set_user(ent, (*parent)->uid, (*parent)->gid);
out:
	return ent;
}
```

<br/>
**tcp4_proc_init_net()-->tcp_proc_register()-->proc_create_data()-->proc_register()**

接着是`proc_register()`，分析如下：

```c
/**
 * @ dir: 传入的是parent，即"/proc/net"
 * @ dp: 传入的是pde，即name为"tcp"的proc_dir_entry
 */
static int proc_register(struct proc_dir_entry * dir, struct proc_dir_entry * dp)
{
	int ret;
	//Return an inode number
	ret = proc_alloc_inum(&dp->low_ino);
	if (ret)
		return ret;
	write_lock(&proc_subdir_lock);
	dp->parent = dir;
	//将新创建的文件添加到其父目录的红黑树中
	if (pde_subdir_insert(dir, dp) == false) {
		WARN(1, "proc_dir_entry '%s/%s' already registered\n",
		     dir->name, dp->name);
		write_unlock(&proc_subdir_lock);
		proc_free_inum(dp->low_ino);
		return -EEXIST;
	}
	write_unlock(&proc_subdir_lock);
	return 0;
}
```

到这里结束`tcp4_proc_init_net()`的分析，可以看到，这里就是将一个name为"tcp"的proc_dir_entry加入到了`"/proc/net/"`这个目录下，而再详细看这个name为"tcp"的proc_dir_entry,如下：

```c
struct proc_dir_entry {
	unsigned int low_ino;  //inode number
	umode_t mode;  // 
	nlink_t nlink; // 1
	kuid_t uid;
	kgid_t gid;
	loff_t size;
	const struct inode_operations *proc_iops;  //proc_file_inode_operations
	const struct file_operations *proc_fops;  //tcp_afinfo_seq_fops
	struct proc_dir_entry *parent;  // "/proc/net/"
	struct rb_root subdir;
	struct rb_node subdir_node;
	void *data;  //tcp4_seq_afinfo
	atomic_t count;		/* use count */
	atomic_t in_use;	/* number of callers into module in progress; */
			/* negative -> it's going away RSN */
	struct completion *pde_unload_completion;
	struct list_head pde_openers;	/* who did ->open, but not ->release */
	spinlock_t pde_unload_lock; /* proc_fops checks and pde_users bumps */
	u8 namelen;
	char name[];  //"tcp"
};
```

##






## 参考

* *[/proc/net/tcp中各项参数说明](http://blog.csdn.net/justlinux2010/article/details/21028797)*
* *[序列文件(seq_file)接口](http://blog.csdn.net/gangyanliang/article/details/7244664)*
* *[seq_file工作机制实例](http://blog.csdn.net/liaokesen168/article/details/49183703)*
* *[proc net arp文件的创建]（http://blog.chinaunix.net/uid-20788636-id-3181318.html）
* *[Linux cat 命令源码剖析](http://blog.csdn.net/xzz_hust/article/details/40896079)*
