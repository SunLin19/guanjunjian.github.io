---
layout:     post
title:      "study5.docker源码阅读之五---daemon端对于container start的处理 "
date:       2017-9-28 20:30:00 
author:     "guanjunjian"
categories: Docker源码阅读
tags:
    - docker_run
    - network
---

* content
{:toc}

> 上一篇介绍了daemon端对container create的处理，这一章将详细介绍daemon端对container start的处理，也就是r.postContainersStart函数
>  
> 源码阅读基于docker [version1.17.05.x](https://github.com/moby/moby/tree/17.05.x)。




## 1. r.postContainersStart()

r.postContainersCreate()的实现位于[moby/api/server/router/container/container_routes.go](https://github.com/moby/moby/blob/17.05.x/api/server/router/container/container_routes.go#L133#L172)，代码的主要内容是：

```go
func (s *containerRouter) postContainersStart(ctx context.Context, w http.ResponseWriter, r *http.Request, vars map[string]string) error {
	//获取hostConfig配置信息
	var hostConfig *container.HostConfig
	checkpoint := r.Form.Get("checkpoint")
	checkpointDir := r.Form.Get("checkpoint-dir")
	//调用ContainerStart进一步启动容器
	err := s.backend.ContainerStart(vars["name"], hostConfig, checkpoint, checkpointDir)
}
```

下面分析`ContainerStart()`函数。

## 2. ContainerStart()

ContainerStart()的实现位于[moby/daemon/start.go](https://github.com/moby/moby/blob/17.05.x/daemon/start.go#L21#L87)，代码的主要内容是：

```go
func (daemon *Daemon) ContainerStart(name string, hostConfig *containertypes.HostConfig, checkpoint string, checkpointDir string) error {
	//可以根据全容器ID、容器名、容器ID前缀获取容器对象
	container, err := daemon.GetContainer(name)
	//调用containerStart进一步启动容器
	daemon.containerStart(container, checkpoint, checkpointDir, true)
}
```

## 3. daemon.containerStart()

daemon.containerStart()的实现位于[moby/daemon/start.go](https://github.com/moby/moby/blob/17.05.x/daemon/start.go#L98#L198)，代码的主要内容是：

```go
func (daemon *Daemon) containerStart(container *container.Container, checkpoint string, checkpointDir string, resetRestartManager bool) (err error) {
	//在创建的时候调用了一次，这次start又mount一次？
	err := daemon.conditionalMountOnStart(container)
	//对网络进行初始化，后面再详细分析
	err := daemon.initializeNetworking(container)
	//调用daemon.containerd.Create()进一步启动，后面再详细分析
	err := daemon.containerd.Create(container.ID, checkpoint, checkpointDir, *spec, container.InitializeStdio, createOptions...)
}
```














其中出现的Config主要目的是基于容器的可移植性信息，与host相互独立，Config 包括容器的基本信息，名字，输入输出流等；非可移植性在 HostConfig 结构体中。

对于`s.backend.ContainerCreate()`，进一步查看代码可以看到，它会调用[moby/api/server/router/container/backend.go](https://github.com/moby/moby/blob/17.05.x/api/server/router/container/backend.go#L36),然后再调用到`daemon.containerCreate(params, false)`。

```go
type stateBackend interface {
	ContainerCreate(config types.ContainerCreateConfig) (container.ContainerCreateCreatedBody, error)
	...
}


func (daemon *Daemon) ContainerCreate(params types.ContainerCreateConfig) (containertypes.ContainerCreateCreatedBody, error) {
	return daemon.containerCreate(params, false)
}
```

下面详细分析daemon.containerCreate(params, false)。





## 结语

本文分析了`postContainersCreate()`，这里主要是生成一个container对象，根据config、hostConfig、networkingConfig进行初始化，再交给下一步启动使用。

## 参考

* *[docker 17 源码分析 docker run container 源码分析二 docker start](http://blog.csdn.net/zhonglinzhang/article/details/76435879)*
