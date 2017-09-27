---
layout:     post
title:      " Linux 存储设备的UUID"
date:       2017-04-09 10:51:00
author:     "Arkingc"
categories: Linux管理
tags:
    - Linux管理
---

* content
{:toc}

>“通用唯一识别码(Universally Unique identifier，简称UUID)；
其目的，是让分布式系统中的所有元素，都能有唯一的辨识信息，而不需要通过中央控制端来做辨识...”




## 存储设备的UUID

在Linux系统中，每个存储设备(分区)有一个对应的UUID，**通过UUID可以定位唯一的存储设备，
但是通过设备名可能定位到不同的存储设备**。

下面详细说明这个问题。

Linux通过**磁盘接口类型**以及**分区类型及顺序**来定义设备名。
如sda1，第一部分——“sda”由磁盘接口类型确定，第二部分——“1”由分区类型及顺序确定。
因此，可以看作“sda”标识了一个磁盘，后面的数字“1”标识这个磁盘上的一个分区。

IDE接口的**磁盘标识由磁盘物理位置决定**。一台有2个IDE接口(IDE1，IDE2)的主机上，磁盘标识如下：
（一个IDE接口可接2个磁盘：主设备master，从设备slave）

| 接口 |    Master  |     Slave     |
| :-----: | :-----------: |  :-----------: |
| IDE1 | /dev/hda | /dev/hdb |
| IDE2 | /dev/hdc | /dev/hdd |

因此，当只有1块磁盘时，会根据磁盘接到的接口和具体位置来决定磁盘标识。

SATA/USB/SCSI接口的磁盘使用SCSI模块来驱动，磁盘标识为/dev/sd[a-p]。
但是，和IDE接口的存储设备不同，**它们跟磁盘的物理位置无关**，设备名由Linux内核检测到的顺序决定。
假设有一块磁盘A接到SATA接口2时，它的磁盘标识为sda。如果在SATA接口1插入一块磁盘B，
磁盘A的标识会变成sdb，因此通过sda1就无法准确定位到正确的存储设备。

## 查看存储设备的UUID

可以通过如下方式查看存储设备的UUID：

```bash
#方法一:
ls -l /dev/disk/by-uuid/
#方法二:
sudo blkid
```

在我的机器上结果如下：


## 参考资料

* *[维基百科](https://zh.wikipedia.org/wiki/%E9%80%9A%E7%94%A8%E5%94%AF%E4%B8%80%E8%AF%86%E5%88%AB%E7%A0%81)*
* *[Linux磁盘分区UUID的获取及其UUID的作用](http://www.cnblogs.com/xia/archive/2011/01/30/1947706.html)*