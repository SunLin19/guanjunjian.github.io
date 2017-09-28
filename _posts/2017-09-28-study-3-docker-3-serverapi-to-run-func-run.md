---
layout:     post
title:      "study3.docker源码阅读之三---从serverapi到docker run的调用 "
date:       2017-9-28 10:00:00 
author:     "guanjunjian"
categories: Docker源码阅读
tags:
    - docker_run
    - network
---

* content
{:toc}

> 上一篇介绍了docker daemon到serverapi的初始化过程，这一篇介绍从serverapi到docker run的调用;
>  
> 上文分析到initRouter(api, d, c)，它初始化了client发来的各种命令的路由，在其中可以追踪到对于create和start命令;
>  
> 源码阅读基于docker [version1.17.05.x](https://github.com/moby/moby/tree/17.05.x)。




## 1. initRouter(api, d, c)路由初始化

### 1.1 源码

initRouter的实现位于[moby/cmd/dockerd/daemon.go](https://github.com/moby/moby/blob/17.05.x/cmd/dockerd/daemon.go#L477#L507)，代码的主要内容是：

```go
func initRouter(s *apiserver.Server, d *daemon.Daemon, c *cluster.Cluster) {
	decoder := runconfig.ContainerDecoder{}//获取解码器
	routers := []router.Router{
		...
		//与container的路由，例如/containers/create,/containers/{name:.*}/start
		container.NewRouter(d, decoder),   
		...
	}
	//如果允许网络控制，则添加network相关的路由
	if d.NetworkControllerEnabled() {
		routers = append(routers, network.NewRouter(d, c))
	}
	//如果是experimental模式将所有路由数据项中的experimental模式下的api路由功能激活
	if d.HasExperimental() {
		for _, r := range routers {
			for _, route := range r.Routes() {
				if experimental, ok := route.(router.ExperimentalRoute); ok {
					experimental.Enable()
				}
			}
		}
	}
	//根据设置好的路由表routers来初始化apiServer的路由器
	s.InitRouter(debug.IsEnabled(), routers...)  
}
```

### 1.2 流程图

![](/img/study/study-3-docker-3-serverapi-to-run-func-3/docker-daemon-initRouter.png)

从图中可以看到，`docker run`发出的`"/containers/create"`和`"/containers/{name:.*}/start"`最后会分别路由到`postContainersCreate`和`postContainersStart`函数，在这两个函数中分别做容器创建和启动的工作，这两个函数将在后面的文章中分析，本文将进一步分析initRouter这个函数，了解清楚路由是如何分发的。
路由初始化过程分为以下三个步骤：

* 向路由表routers中添加路由数据项router
* 检验experimental模式，若为experimental模式则激活该模式下的api路由功能
* 根据路由表初始化apiServer路由器


下面挑选步骤1和步骤3详细讲解。

## 2. 路由初始化步骤分析

### 2.1 路由表添加路由数据

对于docker run命令的路由表添加为`container.NewRouter(d, decoder)`，代码为：
```go
//从上文container.NewRouter(d, decoder)可以看出，传入的b=d，为daemon.Daemon实例
//这里初始化了一个container相关的路由器对象
func NewRouter(b Backend, decoder httputils.ContainerDecoder) router.Router {
	r := &containerRouter{
		backend: b,
		decoder: decoder,
	}
	r.initRoutes()
	return r
}

//初始化路由数据
func (r *containerRouter) initRoutes() {
	r.routes = []router.Route{
		...		
		// POST
		router.NewPostRoute("/containers/create", r.postContainersCreate), 
		router.NewPostRoute("/containers/{name:.*}/start", r.postContainersStart),
		...
	}
}
```

可以从上面的`r.initRoutes()`中看到，container相关的添加了create和start相关的路由，下面先看看`router.NewPostRoute()`函数:

```go
//初始化一个以HTTP POST方式的路由,返回类型为Route
func NewPostRoute(path string, handler httputils.APIFunc) Route {
	return NewRoute("POST", path, handler)
}

//初始化一个local route
func NewRoute(method, path string, handler httputils.APIFunc) Route {
	return localRoute{method, path, handler}
}

type localRoute struct {
	method  string //该路由中方法名
	path    string //该路由中方法所在的路径
	handler httputils.APIFunc //该方法的handler
}

```

以create为例，那么它的localRoute的method="POST",path="/containers/create",handler=r.postContainersCreate，所以当daemon收到client发来的`"/containers/create"`时，会调用`r.postContainersCreate`。

### 2.2 apiServer路由器的初始化

该部分由`s.InitRouter(debug.IsEnabled(), routers...)`完成，代码如下：

```go
func (s *Server) InitRouter(enableProfiler bool, routers ...router.Router) {
	s.routers = append(s.routers, routers...)  //将创建好的路由表信息追加到apiServer对象中的routers。
	m := s.createMux()  //追加后再次初始化apiServer路由器进行更新，下文详细解释
	
	//这里设置好了mux.Route之后，将该route设置到apiServer的路由交换器中去，至此所有deamon.start（）的相关工作处理完毕
	s.routerSwapper = &routerSwapper{
		router: m,
	}  
}
```

下面是`s.createMux()`的具体实现分析：

```go
func (s *Server) createMux() *mux.Router {
	/*
	mux位于vendor/github.com/gorilla/mux,该函数新建一个mux.go中的Route（路由数据项）对象并追加到mux.Router结构体中的成员routes中去，然后返回该路由器mux.Route m
	*/
	m := mux.NewRouter()  
	
	//遍历所有apiserver中的api路由器如：container
	for _, apiRouter := range s.routers {
		//遍历每个apiRouter的子命令路由r如"/containers/create"  
		for _, r := range apiRouter.Routes() {
			//给每个r的路由handler包裹了一层中间件（这里还不是很清楚）
			f := s.makeHTTPHandler(r.Handler())
			/*
			在mux.Route路由结构中根据这个r.Path()路径设置一个适配器来匹配方法method和handler，当满足versionMatcher+r.Path()路径的正则表达式要求就可以适配到相应的方法名及该handler
			*/
			m.Path(versionMatcher + r.Path()).Methods(r.Method()).Handler(f)
			//同上
			m.Path(r.Path()).Methods(r.Method()).Handler(f) 
		}
	}
	...
	return m
}

```

## 结语

本文分析了从apiserver路由到具体的命令执行函数，相对于`docker run`就是到达`r.postContainersCreate`和`r.postContainersStart`，后面的文章会分别对这两个函数详细分析。

## 参考

* *[docker源码阅读笔记四---Xubo](http://blog.xbblfz.site/2017/04/21/docker%E5%AE%A2%E6%88%B7%E7%AB%AF%E4%B8%8E%E6%9C%8D%E5%8A%A1%E5%99%A8%E9%80%9A%E4%BF%A1%E6%A8%A1%E5%9D%97%E4%BA%8C/)*
