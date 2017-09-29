---
layout:     post
title:      "study5.docker源码阅读之五---daemon端对于container start的处理"
date:       2017-9-28 20:30:00 
author:     "guanjunjian"
categories: Docker源码阅读
tags:
    - docker_run
    - network
    - study
---

* content
{:toc}

> 上一篇介绍了daemon端对container create的处理，这一章将详细介绍daemon端对container start的处理，也就是r.postContainersStart函数
>  
> 源码阅读基于docker [version1.17.05.x](https://github.com/moby/moby/tree/17.05.x)。




## 1. r.postContainersStart()

### 1.1 源码
r.postContainersCreate()的实现位于[moby/api/server/router/container/container_routes.go#L133#L172](https://github.com/moby/moby/blob/17.05.x/api/server/router/container/container_routes.go#L133#L172)，代码的主要内容是：

```go
func (s *containerRouter) postContainersStart(ctx context.Context, w http.ResponseWriter, r *http.Request, vars map[string]string) error {
	//获取hostConfig配置信息
	var hostConfig *container.HostConfig
	checkpoint := r.Form.Get("checkpoint")
	checkpointDir := r.Form.Get("checkpoint-dir")
	//调用ContainerStart进一步启动容器，2.详细分析
	err := s.backend.ContainerStart(vars["name"], hostConfig, checkpoint, checkpointDir)
}
```

### 1.2 流程图

![](/img/study/study-5-docker-5-postContainersStart/docker-daemon-postContainersStart.png)

下面2.分析`ContainerStart()`函数。

## 2. ContainerStart()

ContainerStart()的实现位于[moby/daemon/start.go#L21#L87](https://github.com/moby/moby/blob/17.05.x/daemon/start.go#L21#L87)，代码的主要内容是：

```go
func (daemon *Daemon) ContainerStart(name string, hostConfig *containertypes.HostConfig, checkpoint string, checkpointDir string) error {
	//可以根据全容器ID、容器名、容器ID前缀获取容器对象
	container, err := daemon.GetContainer(name)
	//调用containerStart进一步启动容器，3.详细分析
	daemon.containerStart(container, checkpoint, checkpointDir, true)
}
```

下面分析`daemon.containerStart()`函数。

## 3. daemon.containerStart()

daemon.containerStart()的实现位于[moby/daemon/start.go#L98#L198](https://github.com/moby/moby/blob/17.05.x/daemon/start.go#L98#L198)，代码的主要内容是：

```go
func (daemon *Daemon) containerStart(container *container.Container, checkpoint string, checkpointDir string, resetRestartManager bool) (err error) {
	//在创建的时候调用了一次，这次start又mount一次？
	err := daemon.conditionalMountOnStart(container)
	//3.1对网络进行初始化，后面再详细分析
	err := daemon.initializeNetworking(container)
	//调用daemon.containerd.Create()进一步启动，后面4.再详细分析
	err := daemon.containerd.Create(container.ID, checkpoint, checkpointDir, *spec, container.InitializeStdio, createOptions...)
}
```

下面3.1分析`daemon.initializeNetworking(）`函数，4.分析`daemon.containerd.Create()`。

### 3.1 daemon.initializeNetworking()

daemon.initializeNetworking()的实现位于[moby/daemon/container_operations.go#L879#L908](https://github.com/moby/moby/blob/17.05.x/daemon/container_operations.go#L879#L908)。`initalizeNetworking()`对网络进行初始化，docker网络有三种，bridge模式（默认值）、host模式和contaier 模式，根据`config`和`hostConfig`中的参数来确定容器的网络模式，然后调`libnetwork`包来建立网络。代码的主要内容是：

```go
func (daemon *Daemon) initializeNetworking(container *container.Container) error {
	//如果是container模式
	if container.HostConfig.NetworkMode.IsContainer() {
		//获取所要共享网络的container的hosts文件
		nc, err := daemon.getNetworkedContainer(container.ID, container.HostConfig.NetworkMode.ConnectedContainer())
		//将所要共享网络的container的HostnamePath、HostsPath和ResolvConfPath赋值给新container
		initializeNetworkingPaths(container, nc)
		//将所要共享网络的container的Hostname、Domainname赋值给新container
		container.Config.Hostname = nc.Config.Hostname
		container.Config.Domainname = nc.Config.Domainname
		//container模式从这里就返回了，不执行后面的操作
		return nil
	}
	//如果是host模式
	if container.HostConfig.NetworkMode.IsHost() {
		if container.Config.Hostname == "" {
			//将宿主机的hostname赋值给新建container
			container.Config.Hostname, err = os.Hostname()
			//这里host没有返回，回往下继续执行
		}
	}
	//下面3.1.1再详细分析
	err := daemon.allocateNetwork(container)
	//生成container的hosts文件
	return container.BuildHostnameFile()
}
```

下面3.1.1分析`daemon.allocateNetwork()`

#### 3.1.1 daemon.allocateNetwork()到daemon.connectToNetwork()

daemon.allocateNetwork()的实现位于[moby/daemon/container_operations.go#L495#L576](https://github.com/moby/moby/blob/17.05.x/daemon/container_operations.go#L495#L576)，其主要实现为：

```go
func (daemon *Daemon) allocateNetwork(container *container.Container) error {
	//如果没有使用网络，或者是container网络，则直接返回
	if container.Config.NetworkDisabled || container.HostConfig.NetworkMode.IsContainer() {
		return nil
	}
	
	networks := make(map[string]*network.EndpointSettings)
	for n, epConf := range container.NetworkSettings.Networks {
		if n == defaultNetName {
			continue
		}

		networks[n] = epConf
	}		
	for netName, epConf := range networks {
		cleanOperationalData(epConf)
		//下面再详细分析
		if err := daemon.connectToNetwork(container, netName, epConf.EndpointSettings, updateSettings); err != nil {
			return err
		}
	}
	//将hosts配置信息存到disk中
	err := container.WriteHostConfig()
}
```

接着分析`daemon.connectToNetwork()`

```go
func (daemon *Daemon) connectToNetwork(container *container.Container, idOrName string, endpointConfig *networktypes.EndpointSettings, updateSettings bool) (err error) {
	//查找需要连接的网络,n为libnetwork.Network类型，config为*networktypes.NetworkingConfig
	n, config, err := daemon.findAndAttachNetwork(container, idOrName, endpointConfig)
	//获取controller
	controller := daemon.netController
	//获得sandbox
	sb := daemon.getNetworkSandbox(container)
	//从给定的network，也就是n，创建一个endpoint options
	createOptions, err := container.BuildCreateEndpointOptions(n, endpointConfig, sb, daemon.configStore.DNS)
	//创建endpoin
	ep, err := n.CreateEndpoint(endpointName, createOptions...)
	//由network n 创建endpoint Join options
	joinOptions, err := container.BuildJoinOptions(n)
	//将endpoint接入sandbox
	err := ep.Join(sb, joinOptions...)
}
```

对网络创建的分析先到此，后续文章再分析

* 创建network
* connect到network

这两部分。下面先接着把container的start过程分析完。


## 4. daemon.containerd.Create()


daemon.containerd.Create()的实现位于[moby/libcontainerd/client_unix.go#L45#L99](https://github.com/moby/moby/blob/17.05.x/libcontainerd/client_unix.go#L45#L99)，其主要代码为：


```go
func (clnt *client) Create(containerID string, checkpoint string, checkpointDir string, spec specs.Spec, attachStdio StdioCallback, options ...CreateOption) (err error) {
	//生成一个libcontainerd.container对象
	container := clnt.newContainer(filepath.Join(dir, containerID), options...)
	//创建目录
	err := idtools.MkdirAllAs(container.dir, 0700, uid, gid)
	//生成文件config.json
	f, err := os.Create(filepath.Join(container.dir, configFilename))
	//调用container.start()进一步启动container，在5.中继续分析
	return container.start(checkpoint, checkpointDir, attachStdio)
}
```

## 5.container.start()

container.start()的实现位于[moby/libcontainerd/container_unix.go#L93#L175](https://github.com/moby/moby/blob/17.05.x/libcontainerd/container_unix.go#L93#L175)。该函数的主要功能是发送CreateContainerRequest RPC请求给docker-containerd。进程其主要代码为：

```go
func (ctr *container) start(checkpoint string, checkpointDir string, attachStdio StdioCallback) (err error) {
	
	//CreateContainerRequest对象，应该是保存一些容器信息给libcontainer start时使用
	r := &containerd.CreateContainerRequest{
		Id:            ctr.containerID,
		BundlePath:    ctr.dir,
		Stdin:         ctr.fifo(syscall.Stdin),
		Stdout:        ctr.fifo(syscall.Stdout),
		Stderr:        ctr.fifo(syscall.Stderr),
		Checkpoint:    checkpoint,
		CheckpointDir: checkpointDir,
		// check to see if we are running in ramdisk to disable pivot root
		NoPivotRoot: os.Getenv("DOCKER_RAMDISK") != "",
		Runtime:     ctr.runtime,
		RuntimeArgs: ctr.runtimeArgs,
	}	
	//发送CreateContainerRequest RPC请求给docker-containerd，这部分还不是很了解
	resp, err := ctr.client.remote.apiClient.CreateContainer(context.Background(), r)
}
```

目前为止，算是分析完了daemon部分的start过程，接下来的内容应该是交给docker-containerd来继续start，但是目前对containerd还不了解，需要去了解containerd之后，才能继续后面的源码分析。


## 结语

本文分析了daemon对container start的处理做了一个简单分析，很多细节部分没有得到很好的分析，比如网络的具体创建、从daemon到containerd的传递，这些内容将在后续的博客中分析。

## 参考

* *[docker 17 源码分析 docker run container 源码分析二 docker start](http://blog.csdn.net/zhonglinzhang/article/details/76435879)*
* *[Docker1.9后的network](http://blog.csdn.net/halcyonbaby/article/details/50085119)*