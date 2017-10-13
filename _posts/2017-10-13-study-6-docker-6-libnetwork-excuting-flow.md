---
layout:     post
title:      "study6.docker源码阅读之六---网络部分执行流分析（libnetwork源码解读）"
date:       2017-10-13 11:00:00 
author:     "guanjunjian"
categories: Docker源码阅读
tags:
    - docker_run
    - network
    - study
---

* content
{:toc}

> 上一篇分析了daemon对于start的处理，之前的文章是对整个容器启动过程的简要分析，这篇打算libnetwork的执行流程。
>  
> 源码阅读基于docker [version1.17.05.x](https://github.com/moby/moby/tree/17.05.x)。




## 1. libnetwork工作执行流简介

对libnetwork的工作流做一个梗概。

* 指定network的驱动和各项相关参数之后调用 libnetwork.New()创建一个NetWorkController实例。这个实例提供了各种接口，Docker可以通过它创建新的NetWork和Sandbox等；
* 通过controller.NewNetwork(networkType, “network1”)来创建指定类型和名称的Network；
* 通过network.CreateEndpoint（”Endpoint1”）来创建一个Endpoint。在这个函数中Docker为这个Endpoint分配了ip和接口，而对应的network实例中的各项配置信息则会被使用到Endpoint中，其中包括iptables的配置规则和端口信息等；
* 通过调用controller.NewSandbox()来创建Sandbox。这个函数主要调用了namespace和cgroup等来创建一个相对独立的沙盒空间
* 调用ep.Join(sbx)将Endpoint加入指定的Sandbox中，则这个Sandbox也会加入创建Endpoint对应的Network中。


## 2. libnetwork在daemon初始化时的工作

在daemon初始化时，libnetwork的工作主要有两个：

* 在Docker daemon启动的时候，daemon会创建一个bridge驱动所对应的netcontroller，这个controller可以在后面的流程里面对network和sandbox等进行创建;
* 接下来daemon通过调用controller.NewNetwork（）来创建指定类型（bridge类型）和指定名称（即docker0）的network。Libnetwork在接收到创建命令后会使用系统linux的系统调用，创建一个名为docker0的网桥。我们可以在Docker daemon启动后，使用ifconfig命令看到这个名为docker0的网桥。

流程图如下（图中main函数位于[moby/cmd/dockerd/docker.go#100#122](https://github.com/moby/moby/blob/17.05.x/cmd/dockerd/docker.go#L100#L122)）：

![](/img/study/study-6-docker-6-libnetwork-excuting-flow/libnetwork-excute-flow-container-start.png)

## 3. libnetwork在docker创建时的工作

### 3.1 libnetwork的工作主要

在docker创建时，libnetwork的工作主要有：

* 1.在Docker daemon成功启动后，我们就可以使用Docker Client进行容器创建了。容器的网络栈会在容器真正启动之前完成创建。在为容器创建网络栈的时候，首先会取得daemon中的netController，其值就是libnetwork中的向外提供的一组接口，即NetworkController；
* 2.接下来，Docker会调用BuildCreateEndpointOptions（）来创建此容器中endpoint的配置信息。然后再调用CreateEndpoint（）使用上面配置好的信息创建对应的endpoint。在bridge模式中，libnetwork创建的设备是veth pair。Libnetwork中调用netlink.LinkAdd(veth)进行了veth pair的创建，得到的一个veth设备是为了host所准备的，另一个是为了sandbox所准备的。将host端的veth加入到网桥（docker0）中。然后调用netlink.LinkSetUp(host)，启动主机端的veth。最后对endpoint中的端口映射进行配置；
* 3.从本质上来讲，这一部分所做的工作就是调用linux系统调用，进行veth pair的创建。然后将veth pair的一端，作为docker0网桥的一个接口加入到这个网桥中；
* 4.创建SandboxOptions，然后调用controller.NewSandbox()来创建属于此container的新的sandbox。在接收到Docker创建sandbox的请求后，libnetwork会使用系统调用为容器创建一个新的netns，并将这个netns的路径返回给Docker;
* 5.调用ep.Join(sb)将endpoint加入到容器对应的sandbox中。先将endpoint加入到容器对应的sandbox中，然后对endpoint的ip信息和gateway等信息进行配置
* 6.Docker在调用libcontainer来启动容器之后，libcontainer会在容器中的init进程初始化容器坏境的时候，将容器中的所有进程都加入到4.中得到的netns中。这样容器就拥有了属于自己独立的网络栈，进而完成了网络部分的创建和配置工作。

### 3.2 流程图

### 3.3 代码分析

上述工作实现位于allocateNetwork()函数中，该函数位于[moby/daemon/container_operations.go#L495#L576](https://github.com/moby/moby/blob/17.05.x/daemon/container_operations.go#L495#L576)，对于这部分的代码，分析如下：

```go
func (daemon *Daemon) allocateNetwork(container *container.Container) error {
	//取得daemon中的netController
	controller := daemon.netController
	
	/*
		always connect default network first since only default network mode support link and we need do some setting on sandbox initialize for link, but the sandbox only be initialized on first network connecting.
		总是先连接default network，因为只有default network支持link,而我们需要在sandbox初始化时为link做一些设置，而sandbox只在连接第一个网络时初始化
	*/
	//返回daemon使用的默认网络栈
	defaultNetName := runconfig.DefaultDaemonNetworkMode().NetworkName()
	if nConf, ok := container.NetworkSettings.Networks[defaultNetName]; ok {
		//重置操作数据的endpoint设置
		cleanOperationalData(nConf)
		//将sandbox连接到defaultNetwork
		if err := daemon.connectToNetwork(container, defaultNetName, nConf.EndpointSettings, updateSettings); err != nil {
			return err
		}

	}
	
	//将container.NetworkSettings.Networks中的数据取出，存入networks中，这样做是防止connectToNetwork()函数修改container.NetworkSettings.Networks的值
	networks := make(map[string]*network.EndpointSettings)
	for n, epConf := range container.NetworkSettings.Networks {
		if n == defaultNetName {
			continue
		}

		networks[n] = epConf
	}
	
	//遍历container.NetworkSettings.Networks的网络，依次将这些网络加入到container中
	for netName, epConf := range networks {
		cleanOperationalData(epConf)
		if err := daemon.connectToNetwork(container, netName, epConf.EndpointSettings, updateSettings); err != nil {
			return err
		}
	}

	//如果container没有连接到任何网络
	//现在就建立它的sandbox，对于别的情况应该是在connectToNetwork()函数中创建的
	if len(networks) == 0 {
		//如果container还没有sandbox
		if nil == daemon.getNetworkSandbox(container) {
			//为sandbox创建配置信息
			options, err := daemon.buildSandboxOptions(container)
			//为container新建一个sandbox
			sb, err := daemon.netController.NewSandbox(container.ID, options...)
			//更新sandbox的ID和key
			container.UpdateSandboxNetworkSettings(sb)
		}
	}

	//saves the host configuration on disk for the container.
	if err := container.WriteHostConfig(); err != nil {
		return err
	}
	networkActions.WithValues("allocate").UpdateSince(start)
	return nil
}
```

接下来再分析daemon.connectToNetwork(),该函数位于[moby/daemon/container_operations.go#L684#L796](https://github.com/moby/moby/blob/17.05.x/daemon/container_operations.go#L684#L796)，对于这部分的代码，分析如下：

















## 结语

本文分析了daemon对container start的处理做了一个简单分析，很多细节部分没有得到很好的分析，比如网络的具体创建、从daemon到containerd的传递，这些内容将在后续的博客中分析。

## 参考

* *[Docker 网络部分执行流分析（libnetwork源码解读）](http://blog.csdn.net/gao514916467/article/details/51242299)*
