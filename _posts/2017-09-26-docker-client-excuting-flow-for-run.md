---
layout:     post
title:      " docker源码阅读---docker client命令行执行流程 "
date:       2017-9-26 22:40:00 
author:     "guanjunjian"
categories: Docker源码阅读
tags:
    - Docker run
    - network
---

* content
{:toc}

> 开始阅读docker源码，最终目的是了解docker network的实现。
> 
> 源码阅读基于docker [Version1.17.05.x](https://github.com/moby/moby/tree/17.05.x)

## 1.docker client的入口

### 1.1源码

docker client的main函数位于[moby/cmd/docker/docker.go](https://github.com/moby/moby/blob/17.05.x/cmd/docker/docker.go#L161#184)，代码的主要内容是：

<pre><code>func main() {
	...
	dockerCli := command.NewDockerCli(stdin, stdout, stderr)
	cmd := newDockerCommand(dockerCli)
	if err := cmd.Execute(); 
	...
	}
}
</code></pre>

**这部分代码的主要工作是：**

* **1.** 生成一个带有输入输出的客户端对象

* **2.** 根据dockerCli客户端对象，解析命令行参数，生成带有命令行参数及客户端配置信息的cmd命令行对象

* **3.** 根据输入参数args完成命令执行

### 1.2流程图

![](/img/in-post/post-docker-client-excuting-flow-for-run/docker-client-main.png)


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