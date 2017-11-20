---
layout:     post
title:      "HTB源码分析"
date:       2017-11-20 11:00:00 
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
> 参考[Linux内核中流量控制](http://cxw06023273.iteye.com/blog/867318)系列文章。
>   
> 内核代码版本为2.6.19.2
>




## 1. 输出流控

图1.输出流控流程：
![](/img/study/study-9-hierachical-token-bucket-source-code-1/1-egress-qdisc.png)


## 参考

* *[ux内核中流量控制(1)](http://cxw06023273.iteye.com/blog/867318)*

