---
layout:     post
title:      " 从一个实例来看docker的存储驱动——aufs "
date:       2017-04-13 10:51:00
author:     "ArKing"
categories: Docker
tags:
    - Docker存储驱动
    - AUFS
---

* content
{:toc}

> docker支持的存储驱动有aufs、overlayFS、devicemapper、btrfs、zfs;
> 
> 这篇文章主要谈谈aufs；




## 一个aufs的实例

AUFS 是一种 Union File System（联合文件系统），它可以把不同位置的目录合并mount到同一目录下。
其中，最上层目录**读写**(r/w)，其余目录**只读**(read only)。

创建两个目录aaa/，以及bbb/，在目录下创建文件，其结构如下：

![](/img/in-post/post-docker-filesystem/aufs-sample1.png)

文件的内容如下:

![](/img/in-post/post-docker-filesystem/aufs-sample2.png)

通过aufs将aaa/和bbb/的内容mount到mnt目录下。**从容器角度来看**，**aaa/属于读写层**，**bbb/属于只读层**。
aufs根据权限(如果指定了)首先mount bbb/，然后mount aaa/：

![](/img/in-post/post-docker-filesystem/aufs-sample3.png)

整个的结构看起来像下面这样：

![](/img/in-post/post-docker-filesystem/sample-layers.png)

读写层(aaa/)**反应了在该层文件的改动**,也就是说，mount aaa/后，修改了file2(因为copy-on-write,所以bbb/中的file2并没有修改)，新增了file1。
file3的内容为只读层(bbb/)的映射。所以，mnt/file1和mnt/file2应该是aaa/中file1和file2的内容，
mnt/file3为bbb/file3的内容。cat一下看看：

![](/img/in-post/post-docker-filesystem/aufs-sample4.png)

 结果和预期一样。

### 修改读写层(aaa/)的文件

对读写层(aaa/)中的file1和file2进行修改：

![](/img/in-post/post-docker-filesystem/aufs-sample5.png)

由于aaa/为**顶层读写层**，因此对file1和file2进行修改将会**直接修改aaa/中file1和file2内容**。不会影响bbb/中的内容。

### 修改一个只读层(bbb/)的文件

然后试图修改只读层(bbb/)中的file3:

![](/img/in-post/post-docker-filesystem/aufs-sample6.png)

由于file3为只读，因此会触发copy-on-write，在读写层aaa/中创建一个新的file3。通过上图发现，aaa/目录下多出了一个file3,在将bbb/file3拷贝至aaa/中后，再对其进行修改。bbb/file3的内容没有改变。

### 删除一个只读层(bbb/)的文件

那删除一个只读层的文件又如何实现？aufs使用whiteout机制，
通过在上层读写层下创建相应的whiteout隐藏文件来实现删除，本质上是隐藏。rm file3看看：

![](/img/in-post/post-docker-filesystem/aufs-sample7.png)

 aaa/中新增的.wh.file3文件实现了对file3的隐藏，bbb/file3并没有被删除。

## 再来看看docker container

![](/img/in-post/post-docker-filesystem/aufs-layers.jpg)

*[图片来源: Docker文档][i1]*

上图是一个docker容器的层次化结构，docker容器由镜像生成，镜像由多层的**只读的镜像层**组成。每run一个docker容器时，会根据使用的镜像，在只读层上创建一层**读写层(容器层)**。  
从这个图中看，镜像层和容器层对应/var/lib/docker/aufs/diff/下的一系列目录，通过aufs，这些层被mount到/var/lib/docker/aufs/mnt/下的一个目录中。

用ubuntu镜像run了一个container，查看挂在信息：

![](/img/in-post/post-docker-filesystem/container-layers.png)

最后的权限看出，7bf3d..../为容器层，下面的都是镜像层。

进一步查看这些层中的内容：

![](/img/in-post/post-docker-filesystem/container-diff.png)

每一层都给显示了该层的改动，通过aufs，将这些层下的内容mount到一个目录下，形成了最终我们在容器中ls看到的内容。

### 对容器层(读写层)进行修改

以创建一个100M新文件file为例子，观察对容器层进行修改后，容器层怎么记录这个变化：

![](/img/in-post/post-docker-filesystem/container-addfile.png)

容器层中多出了一个file文件。

### 对镜像层(只读层)进行修改

通过删除镜像层的一个目录/home，观察修改后，容器层怎么记录这个变化：

![](/img/in-post/post-docker-filesystem/container-rmfile.png)

是不是和1.3一样？容器层下多出了一个文件.wh.home，实现了对/home目录的隐藏。

## 结语

docker使用aufs作为存储驱动时,通过将**只读的镜像层**和**读写的容器层**mount到同一目录下，将多层合并成文件系统的单层表示。所有的变化都只是记录在容器层，因此它的优势很明显：
* *所有使用相同镜像进行构建的容器，只是在相同镜像上创建了一个容器层，因此能够节省存储资源，同时加速容器的创建和启动时间；*
* *同时，由于aufs出现较早，因此比较稳定，在大量生产中实践过，有较强的社区支持；*  

但是它也存在缺陷：
* *没有并入Linux内核，只支持ubuntu；*
* *它是文件级的存储，对底层大文件的修改，引发的copy-on-write会带来很大的I/O开销；*

## 参考

* *[Docker存储-Aufs](http://www.cnblogs.com/sammyliu/p/5931383.html)*
* *[Docker五种存储驱动原理及应用场景和性能测试对比](http://dockone.io/article/1513)*

[i1]: https://docs.docker.com/engine/userguide/storagedriver/images/aufs_layers.jpg