---
layout:     post
title:      "分层令牌桶原理---<<Hierachical token bucket theory>>翻译"
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

* Class：与其有关的有，假设速率（assured rate AR），最高速率(ceil rate CR),优先级






## 参考

* *[/proc/net/tcp中各项参数说明](http://blog.csdn.net/justlinux2010/article/details/21028797)*
* *[序列文件(seq_file)接口](http://blog.csdn.net/gangyanliang/article/details/7244664)*
* *[seq_file工作机制实例](http://blog.csdn.net/liaokesen168/article/details/49183703)*
* *[proc net arp文件的创建](http://blog.chinaunix.net/uid-20788636-id-3181318.html)*
* *[Linux cat 命令源码剖析](http://blog.csdn.net/xzz_hust/article/details/40896079)*
* *[linux文件系统之读流程 SYSCALL_DEFINE3(read, xxx)](http://blog.csdn.net/yuzhihui_no1/article/details/51298498)*
* *[proc_dir_entry结构](http://blog.sina.com.cn/s/blog_70441c8e0102wex8.html)*
* *[linux下proc文件的读写(部分转载)](http://blog.csdn.net/hunanchenxingyu/article/details/8102956)*
* *[seq_file文件的内核读取过程](https://www.cnblogs.com/Wandererzj/archive/2012/04/16/2452209.html)*
* *[linux 读取proc文件之seq_file浅析1](http://blog.csdn.net/zhanshenwu/article/details/24555323)*
* *[kernel module编程（八）：读取proc文件之seq_file](http://blog.csdn.net/flowingflying/article/details/4566701)*
* *[走马观花： Linux 系统调用 open 七日游（一）](http://blog.chinaunix.net/uid-20522771-id-4419666.html)*
