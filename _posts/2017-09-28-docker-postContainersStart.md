---
layout:     post
title:      " docker源码阅读之五---daemon端对于container start的处理 "
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

r.postContainersCreate()的实现位于[moby/cmd/dockerd/daemon.go](https://github.com/moby/moby/blob/17.05.x/api/server/router/container/container_routes.go#L133#L172)，代码的主要内容是：

```go
func (s *containerRouter) postContainersCreate(ctx context.Context, w http.ResponseWriter, r *http.Request, vars map[string]string) error {
	//从http的form表单中获取名字，应该就是"/containers/create"吧？
	name := r.Form.Get("name")
	//获取从client传过来的Config、hostConfig和networkingConfig配置信息
	config, hostConfig, networkingConfig, err := s.decoder.DecodeConfig(r.Body)
	if err != nil {
		return err
	}
	//传入配置信息，调用ContainerCreate进一步创建容器
	ccr, err := s.backend.ContainerCreate(types.ContainerCreateConfig{
		Name:             name,
		Config:           config,
		HostConfig:       hostConfig,
		NetworkingConfig: networkingConfig,
		AdjustCPUShares:  adjustCPUShares,
	})
	
	//给client返回结果
	return httputils.WriteJSON(w, http.StatusCreated, ccr)
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

* *[docker 17 源码分析 docker run container 源码分析一 docker create](http://blog.csdn.net/zhonglinzhang/article/details/53435590)*
