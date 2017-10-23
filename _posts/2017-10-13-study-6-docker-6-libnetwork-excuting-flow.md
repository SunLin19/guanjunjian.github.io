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

> 上一篇分析了daemon对于start的处理，之前的文章是对整个容器启动过程的简要分析，这篇分析libnetwork的执行流程。
>  
> 源码阅读基于docker [version1.17.05.x](https://github.com/moby/moby/tree/17.05.x)。




## 1. libnetwork工作执行流简介

对libnetwork的工作流做一个梗概。

* 指定network的驱动和各项相关参数之后调用 libnetwork.New()创建一个NetWorkController实例。这个实例提供了各种接口，Docker可以通过它创建新的NetWork和Sandbox等；
* 通过controller.NewNetwork(networkType, “network1”)来创建指定类型和名称的Network；
* 通过network.CreateEndpoint（”Endpoint1”）来创建一个Endpoint。在这个函数中Docker为这个Endpoint分配了ip和接口，而对应的network实例中的各项配置信息则会被使用到Endpoint中，其中包括iptables的配置规则和端口信息等；
* 通过调用controller.NewSandbox()来创建Sandbox。这个函数主要调用了namespace和cgroup等来创建一个相对独立的沙盒空间
* 调用ep.Join(sbx)将Endpoint加入指定的Sandbox中，则这个Sandbox也会加入创建Endpoint对应的Network中。

而上面的这部分工作分别在daemon初始化和docker创建时执行，所以下面分别按这两方面再详细分析。


## 2. libnetwork在daemon初始化时的工作

### 2.1 libnetwork的工作

在daemon初始化时，libnetwork的工作主要有两个：

* 在Docker daemon启动的时候，daemon会创建一个bridge驱动所对应的netcontroller，这个controller可以在后面的流程里面对network和sandbox等进行创建;
* 接下来daemon通过调用controller.NewNetwork（）来创建指定类型（bridge类型）和指定名称（即docker0）的network。Libnetwork在接收到创建命令后会使用系统linux的系统调用，创建一个名为docker0的网桥。我们可以在Docker daemon启动后，使用ifconfig命令看到这个名为docker0的网桥。

### 2.2 流程图

流程图如下（图中main函数位于[moby/cmd/dockerd/docker.go#100#122](https://github.com/moby/moby/blob/17.05.x/cmd/dockerd/docker.go#L100#L122)）：

![](/img/study/study-6-docker-6-libnetwork-excuting-flow/libnetwork-excute-flow-daemon-init.png)

### 2.3 代码分析

这部分分析从`daemon.netController, err = daemon.initNetworkController(daemon.configStore, activeSandboxes)`开始，`daemon.initNetworkController()`的实现位于[moby/daemon/daemon_unix.go#L727#L774](https://github.com/moby/moby/blob/17.05.x/daemon/daemon_unix.go#L727#L774)，对于这部分的代码，分析如下：

```go
func (daemon *Daemon) initNetworkController(config *config.Config, activeSandboxes map[string]interface{}) (libnetwork.NetworkController, error) {
	//network options,如OptionExecRoot、OptionDefaultDriver、OptionDefaultNetwork等
	netOptions, err := daemon.networkOptions(config, daemon.PluginStore, activeSandboxes)
	//根据netOptions创建net controller，2.3.1分析
	controller, err := libnetwork.New(netOptions...)
	// Initialize default network on "null"
	if n, _ := controller.NetworkByName("none"); n == nil {
		//如果没有就新建一个，NewNetwork()在2.3.2分析
		if _, err := controller.NewNetwork("null", "none", "", libnetwork.NetworkOptionPersist(true)); err != nil {
			return nil, fmt.Errorf("Error creating default \"null\" network: %v", err)
		}
	}
	// Initialize default network on "host"
	if n, _ := controller.NetworkByName("host"); n == nil {
		if _, err := controller.NewNetwork("host", "host", "", libnetwork.NetworkOptionPersist(true)); err != nil {
			return nil, fmt.Errorf("Error creating default \"host\" network: %v", err)
		}
	}
	// Clear stale bridge network
	if n, err := controller.NetworkByName("bridge"); err == nil {
		if err = n.Delete(); err != nil {
			return nil, fmt.Errorf("could not delete the default bridge network: %v", err)
		}
	}
	if !config.DisableBridge {
		// Initialize default driver "bridge"，在2.3.3分析
		if err := initBridgeDriver(controller, config); err != nil {
			return nil, err
		}
	} else {
		removeDefaultBridgeInterface()
	}
	return controller, nil
}
```

#### 2.3.1 libnetwork.New()

`libnetwork.New(netOptions...)`的实现位于[moby/vendor/github.com/docker/libnetwork/controller.go#L179#L244](https://github.com/moby/moby/blob/17.05.x/vendor/github.com/docker/libnetwork/controller.go#L179#L244)，对于这部分的代码，分析如下：

```go
func New(cfgOptions ...config.Option) (NetworkController, error) {
	c := &controller{
		id:              stringid.GenerateRandomID(),
		cfg:             config.ParseConfigOptions(cfgOptions...),
		sandboxes:       sandboxTable{},
		svcRecords:      make(map[string]svcInfo),
		serviceBindings: make(map[serviceKey]*service),
		agentInitDone:   make(chan struct{}),
		networkLocker:   locker.New(),
	}
	c.initStores()
	//DrvRegistry holds the registry of all network drivers and IPAM drivers that it knows about.
	drvRegistry, err := drvregistry.New(c.getStore(datastore.LocalScope), c.getStore(datastore.GlobalScope), c.RegisterDriver, nil, c.cfg.PluginGetter)
	/*the daemon are enabled for the experimental features，这是一个bool变量，true的话会增加额外的驱动初始化结构体，如ipvlan
	这个函数调用的是./libnetwork/drivers_linux.go，返回的是bridge、host、macvlan、null、remote、overlay的初始化结构体initializer数组，这个结构体包含了驱动的初始化函数和驱动的名字,以bridge驱动为例：{bridge.Init, "bridge"}
	在2.3.1.1中分析bridge的初始化过程
	*/
	for _, i := range getInitializers(c.cfg.Daemon.Experimental) {
		var dcfg map[string]interface{}

		// External plugins don't need config passed through daemon. They can
		// bootstrap themselves
		//如果不是remote驱动，则创建driver config
		if i.ntype != "remote" {
			dcfg = c.makeDriverConfig(i.ntype)
		}
		//将这些驱动的名字、初始化函数、config添加到drvRegistry中
		if err := drvRegistry.AddDriver(i.ntype, i.fn, dcfg); err != nil {
			return nil, err
		}
	}
	//初始化IPAMDrivers
	initIPAMDrivers(drvRegistry, nil, c.getStore(datastore.GlobalScope))
	//赋值controller的drvRegistry
	c.drvRegistry = drvRegistry

	if c.cfg != nil && c.cfg.Cluster.Watcher != nil {
		if err := c.initDiscovery(c.cfg.Cluster.Watcher); err != nil {
			// Failing to initialize discovery is a bad situation to be in.
			// But it cannot fail creating the Controller
			logrus.Errorf("Failed to Initialize Discovery : %v", err)
		}
	}
	//遍历controller所有network，如果network has special drivers, which do not need to perform any network plumbing,将添加到controller中
	c.WalkNetworks(populateSpecial)
	/*
	Reserve pools first before doing cleanup. Otherwise the cleanups of endpoint/network and sandbox below will generate many unnecessary warnings
	*/
	c.reservePools()
	// Cleanup resources
	c.sandboxCleanup(c.cfg.ActiveSandboxes)
	c.cleanupLocalEndpoints()
	c.networkCleanup()
	// /run/docker/libnetwork/container ID.sock开始监听，2.3.1.2分析
	c.startExternalKeyListener()
	return c, nil
}
```

##### 2.3.1.1 bridge驱动的初始化

从`for _, i := range getInitializers(c.cfg.Daemon.Experimental)`中有`{bridge.Init, "bridge"}`，因此bridge驱动的初始化函数为bridge.Init()，它的实现位于[moby/vendor/github.com/docker/libnetwork/drivers/bridge/bridge.go#L149#L159](https://github.com/moby/moby/blob/17.05.x/vendor/github.com/docker/libnetwork/drivers/bridge/bridge.go#L149#L159)，代码分析如下：

```go
func Init(dc driverapi.DriverCallback, config map[string]interface{}) error {
	//在下面详细看
	d := newDriver()
	if err := d.configure(config); err != nil {
		return err
	}
	// Capability represents the high level capabilities of the drivers which libnetwork can make use of
	c := driverapi.Capability{
		DataScope: datastore.LocalScope,
	}
	//RegisterDriver provides a way for Remote drivers to dynamically register new NetworkType and associate with a driver instance
	return dc.RegisterDriver(networkType, d, c)
}
```

接着看看`newDriver()`，它的实现位于[moby/vendor/github.com/docker/libnetwork/drivers/bridge/bridge.go#L144#L146](https://github.com/moby/moby/blob/17.05.x/vendor/github.com/docker/libnetwork/drivers/bridge/bridge.go#L144#L146)，代码分析如下：

```go
func newDriver() *driver {
	//返回一个driver结构体
	return &driver{networks: map[string]*bridgeNetwork{}, config: &configuration{}}
}
```

再继续看看driver结构体

```go
type driver struct {
	// configuration info for the "bridge" driver.
	config         *configuration
	network        *bridgeNetwork
	// ChainInfo defines the iptables chain.
	natChain       *iptables.ChainInfo
	filterChain    *iptables.ChainInfo
	isolationChain *iptables.ChainInfo
	networks       map[string]*bridgeNetwork
	store          datastore.DataStore
	/*
	Handle is an handle for the netlink requests on a specific network namespace. All the requests on the same netlink family share the same netlink socket,which gets released when the handle is deleted.
	*/
	nlh            *netlink.Handle
	sync.Mutex
}
```


##### 2.3.1.2 c.startExternalKeyListener()

`c.startExternalKeyListener()`实现位于[moby/vendor/github.com/docker/libnetwork/sandbox_externalkey_unix.go#L105#L124](https://github.com/moby/moby/blob/17.05.x/vendor/github.com/docker/libnetwork/sandbox_externalkey_unix.go#L105#L124)，对于这部分的代码，分析如下：

```go
func (c *controller) startExternalKeyListener() error {
	//udsBase = "/run/docker/libnetwork/"，这里创建"/run/docker/libnetwork/"目录，访问权限0600
	os.MkdirAll(udsBase, 0600)
	//uds="/run/docker/libnetwork/"+c.id+".sock"
	uds := udsBase + c.id + ".sock"
	//监听这个sock
	l, err := net.Listen("unix", uds)
	os.Chmod(uds, 0600)
	c.Lock()
	c.extKeyListener = l
	c.Unlock()
	//新起一个routine，接收客户端连接
	go c.acceptClientConnections(uds, l)
	return nil
}
```


#### 2.3.2 controller.NewNetwork()

#### 2.3.3 initBridgeDriver()



## 3. libnetwork在docker创建时的工作

### 3.1 libnetwork的工作

在docker创建时，libnetwork的工作主要有：

* 1.在Docker daemon成功启动后，我们就可以使用Docker Client进行容器创建了。容器的网络栈会在容器真正启动之前完成创建。在为容器创建网络栈的时候，首先会取得daemon中的netController，其值就是libnetwork中的向外提供的一组接口，即NetworkController；
* 2.接下来，Docker会调用BuildCreateEndpointOptions（）来创建此容器中endpoint的配置信息。然后再调用CreateEndpoint（）使用上面配置好的信息创建对应的endpoint。在bridge模式中，libnetwork创建的设备是veth pair。Libnetwork中调用netlink.LinkAdd(veth)进行了veth pair的创建，得到的一个veth设备是为了host所准备的，另一个是为了sandbox所准备的。将host端的veth加入到网桥（docker0）中。然后调用netlink.LinkSetUp(host)，启动主机端的veth。最后对endpoint中的端口映射进行配置；
* 3.从本质上来讲，这一部分所做的工作就是调用linux系统调用，进行veth pair的创建。然后将veth pair的一端，作为docker0网桥的一个接口加入到这个网桥中；
* 4.创建SandboxOptions，然后调用controller.NewSandbox()来创建属于此container的新的sandbox。在接收到Docker创建sandbox的请求后，libnetwork会使用系统调用为容器创建一个新的netns，并将这个netns的路径返回给Docker;
* 5.调用ep.Join(sb)将endpoint加入到容器对应的sandbox中。先将endpoint加入到容器对应的sandbox中，然后对endpoint的ip信息和gateway等信息进行配置
* 6.Docker在调用libcontainer来启动容器之后，libcontainer会在容器中的init进程初始化容器坏境的时候，将容器中的所有进程都加入到4.中得到的netns中。这样容器就拥有了属于自己独立的网络栈，进而完成了网络部分的创建和配置工作。

### 3.2 流程图

![](/img/study/study-6-docker-6-libnetwork-excuting-flow/libnetwork-excute-flow-container-start.png)

### 3.3 代码分析

#### 3.3.1 allocateNetwork()

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

#### 3.3.2 connectToNetwork()

接下来再分析daemon.connectToNetwork(),该函数位于[moby/daemon/container_operations.go#L684#L796](https://github.com/moby/moby/blob/17.05.x/daemon/container_operations.go#L684#L796)，对于这部分的代码，分析如下：

```go
//以daemon.connectToNetwork(container, netName, epConf.EndpointSettings, updateSettings)为例
func (daemon *Daemon) connectToNetwork(container *container.Container, idOrName string, endpointConfig *networktypes.EndpointSettings, updateSettings bool) (err error) {
	//如果是container模式，返回错误
	if container.HostConfig.NetworkMode.IsContainer() {
		return runconfig.ErrConflictSharedNetwork
	}
	//根据network的name或id查找network，并连接，在3.3.3中再详细分析
	n, config, err := daemon.findAndAttachNetwork(container, idOrName, endpointConfig)
	
	var operIPAM bool
	if config != nil {
		if epConfig, ok := config.EndpointsConfig[n.Name()]; ok {
			if endpointConfig.IPAMConfig == nil ||
				(endpointConfig.IPAMConfig.IPv4Address == "" &&
					endpointConfig.IPAMConfig.IPv6Address == "" &&
					len(endpointConfig.IPAMConfig.LinkLocalIPs) == 0) {
				operIPAM = true
			}
			// 将连接的network的epConfig的IPAMConcif和NetworkID赋值到endpointConfig中，用于之后的网络配置更新
			endpointConfig.IPAMConfig = epConfig.IPAMConfig
			endpointConfig.NetworkID = epConfig.NetworkID
		}
	}
	//更新容器的网络配置
	err = daemon.updateNetworkConfig(container, n, endpointConfig, updateSettings)
	//获得netController
	controller := daemon.netController
	//获得容器的sandbox,从controller的Sandbox数组中获取,在3.3.4中分析
	sb := daemon.getNetworkSandbox(container)
	//创建endpoint的配置信息
	createOptions, err := container.BuildCreateEndpointOptions(n, endpointConfig, sb, daemon.configStore.DNS)
	//endPoint的名字
	endpointName := strings.TrimPrefix(container.Name, "/")
	//创建endpoint,下面在3.3.5中分析
	ep, err := n.CreateEndpoint(endpointName, createOptions...)
	//将network的endpoint信息赋值到NetworkSettings中
	container.NetworkSettings.Networks[n.Name()] = &network.EndpointSettings{
		EndpointSettings: endpointConfig,
		IPAMOperational:  operIPAM,
	}
	
	if _, ok := container.NetworkSettings.Networks[n.ID()]; ok {
		delete(container.NetworkSettings.Networks, n.ID())
	}
	//更新Endpoint信息
	if err := daemon.updateEndpointNetworkSettings(container, n, ep); err != nil {
		return err
	}
	//如果sandbox为空，则新建一个
	if sb == nil {
		options, err := daemon.buildSandboxOptions(container)
		if err != nil {
			return err
		}
		//在3.3.4中分析
		sb, err = controller.NewSandbox(container.ID, options...)
		if err != nil {
			return err
		}

		container.UpdateSandboxNetworkSettings(sb)
	}
	//根据network创建endpoint join的配置信息
	joinOptions, err := container.BuildJoinOptions(n)
	//将endpoint加入sandbox中，将在3.3.6分析
	ep.Join(sb, joinOptions...)
	//start中传入的managed为false
	if !container.Managed {
		// add container name/alias to DNS
		if err := daemon.ActivateContainerServiceBinding(container.Name); err != nil {
			return fmt.Errorf("Activate container service binding for %s failed: %v", container.Name, err)
		}
	}
	//当container加入endpoint时，更新network的配置
	if err := container.UpdateJoinInfo(n, ep); err != nil {
		return fmt.Errorf("Updating join info failed: %v", err)
	}
	//这里调用的是container.GetSandboxPortMapInfo，返回sandbox的端口映射情况
	container.NetworkSettings.Ports = getPortMapInfo(sb)

	daemon.LogNetworkEventWithAttributes(n, "connect", map[string]string{"container": container.ID})
	networkActions.WithValues("connect").UpdateSince(start)
	return nil
}
```

#### 3.3.3 network---findAndAttachNetwork()

接下来再分析daemon.findAndAttachNetwork()。

流程图如下：
![](/img/study/study-6-docker-6-libnetwork-excuting-flow/libnetwork-excute-flow-container-start-findAndAttachNetwork.png)

该函数位于[moby/daemon/container_operations.go#L341#L421](https://github.com/moby/moby/blob/17.05.x/daemon/container_operations.go#L341#L421)，对于这部分的代码，分析如下：

```go
func (daemon *Daemon) findAndAttachNetwork(container *container.Container, idOrName string, epConfig *networktypes.EndpointSettings) (libnetwork.Network, *networktypes.NetworkingConfig, error) {
	//根据name或者id查找network
	n, err := daemon.FindNetwork(idOrName)
	// If we found a network and if it is not dynamically created
	// we should never attempt to attach to that network here.
	if n != nil {
		if container.Managed || !n.Info().Dynamic() {
			return n, nil, nil
		}
	}
	//将IPAMConfig中的IPv4和IPv6的地址添加到addresses数组中
	var addresses []string
	if epConfig != nil && epConfig.IPAMConfig != nil {
		if epConfig.IPAMConfig.IPv4Address != "" {
			addresses = append(addresses, epConfig.IPAMConfig.IPv4Address)
		}

		if epConfig.IPAMConfig.IPv6Address != "" {
			addresses = append(addresses, epConfig.IPAMConfig.IPv6Address)
		}
	}
	var (
		config     *networktypes.NetworkingConfig
		retryCount int
	)
	//这段对cluster网络的处理
	for {
		// In all other cases, attempt to attach to the network to
		// trigger attachment in the swarm cluster manager.  
		//其他情况下，尝试连接到swarm cluster manager的网络触发附件中
		//？？？这端不是很懂
		//对cluster网络的处理
		if daemon.clusterProvider != nil {
			var err error
			config, err = daemon.clusterProvider.AttachNetwork(idOrName, container.ID, addresses)
			if err != nil {
				return nil, nil, err
			}
		}
		n, err = daemon.FindNetwork(idOrName)
		if err != nil {
			if daemon.clusterProvider != nil {
				if err := daemon.clusterProvider.DetachNetwork(idOrName, container.ID); err != nil {
					logrus.Warnf("Could not rollback attachment for container %s to network %s: %v", container.ID, idOrName, err)
				}
			}

			// Retry network attach again if we failed to
			// find the network after successful
			// attachment because the only reason that
			// would happen is if some other container
			// attached to the swarm scope network went down
			// and removed the network while we were in
			// the process of attaching.
			if config != nil {
				if _, ok := err.(libnetwork.ErrNoSuchNetwork); ok {
					if retryCount >= 5 {
						return nil, nil, fmt.Errorf("could not find network %s after successful attachment", idOrName)
					}
					retryCount++
					continue
				}
			}

			return nil, nil, err
		}

		break
	}
	// This container has attachment to a swarm scope
	// network. Update the container network settings accordingly.
	container.NetworkSettings.HasSwarmEndpoint = true
	return n, config, nil
}	
```

接下来看看`daemon.FindNetwork(idOrName)`，代码位于[moby/daemon/network.go#L33#L46](https://github.com/moby/moby/blob/17.05.x/daemon/network.go#L33#L46)，主要代码如下：

```go
func (daemon *Daemon) FindNetwork(idName string) (libnetwork.Network, error) {
	//根据name查找
	n, err := daemon.GetNetworkByName(idName)
	if n != nil {
		return n, nil
	}
	//根据id查找
	return daemon.GetNetworkByID(idName)
}
```

再继续看`daemon.GetNetworkByName(idName)`，代码位于[moby/daemon/network.go#L69#L78](https://github.com/moby/moby/blob/17.05.x/daemon/network.go#L69#L78)，该函数根据name获取network,如果没有对应名字的network，则返回默认network,主要代码为：
```go
func (daemon *Daemon) GetNetworkByName(name string) (libnetwork.Network, error) {
	//获得controller
	c := daemon.netController
	if name == "" {
		name = c.Config().Daemon.DefaultNetwork
	}
	return c.NetworkByName(name)
}
```

接着看`c.NetworkByName(name)`，代码位于[moby/vendor/github.com/docker/libnetwork/controller.go#L892#L913](https://github.com/moby/moby/blob/17.05.x/vendor/github.com/docker/libnetwork/controller.go#L892#L913)，主要代码为：

```go
func (c *controller) NetworkByName(name string) (Network, error) {
	var n Network
	//查找network时，确认是所查network用到的函数
	s := func(current Network) bool {
		if current.Name() == name {
			n = current
			return true
		}
		return false
	}
	遍历controller中的network
	c.WalkNetworks(s)
	return n, nil
}
```

接下来，`c.WalkNetworks(s)`，代码位于[moby/vendor/github.com/docker/libnetwork/controller.go#L884#L890](https://github.com/moby/moby/blob/17.05.x/vendor/github.com/docker/libnetwork/controller.go#L884#L890)，主要代码为：

```go
func (c *controller) WalkNetworks(walker NetworkWalker) {
	for _, n := range c.Networks() {
		if walker(n) {
			return
		}
	}
}
```

再看看`c.Networks()`，代码位于[moby/vendor/github.com/docker/libnetwork/controller.go#L866#L882](https://github.com/moby/moby/blob/17.05.x/vendor/github.com/docker/libnetwork/controller.go#L866#L882)，主要代码为：

```go
func (c *controller) Networks() []Network {
	var list []Network
	networks, err := c.getNetworksFromStore()
	for _, n := range networks {
		if n.inDelete {
			continue
		}
		list = append(list, n)
	}
	return list
}
```

接着`c.getNetworksFromStore()`，代码位于[moby/vendor/github.com/docker/libnetwork/store.go#L142#L181](https://github.com/moby/moby/blob/17.05.x/vendor/github.com/docker/libnetwork/store.go#L142#L181)，主要代码为：

```go
func (c *controller) getNetworksFromStore() ([]*network, error) {
	var nl []*network
	//遍历controller的stores,类型为[]datastore.DataStore
	for _, store := range c.getStores() {
		kvol, err := store.List(datastore.Key(datastore.NetworkKeyPrefix),
			&network{ctrlr: c})
		// Continue searching in the next store if no keys found in this store
		if err != nil {
			if err != datastore.ErrKeyNotFound {
				logrus.Debugf("failed to get networks for scope %s: %v", store.Scope(), err)
			}
			continue
		}

		kvep, err := store.Map(datastore.Key(epCntKeyPrefix), &endpointCnt{})
		if err != nil {
			if err != datastore.ErrKeyNotFound {
				logrus.Warnf("failed to get endpoint_count map for scope %s: %v", store.Scope(), err)
			}
		}

		for _, kvo := range kvol {
			n := kvo.(*network)
			n.Lock()
			n.ctrlr = c
			ec := &endpointCnt{n: n}
			// Trim the leading & trailing "/" to make it consistent across all stores
			if val, ok := kvep[strings.Trim(datastore.Key(ec.Key()...), "/")]; ok {
				ec = val.(*endpointCnt)
				ec.n = n
				n.epCnt = ec
			}
			n.scope = store.Scope()
			n.Unlock()
			nl = append(nl, n)
		}
	}
	return nl, nil
}
```

最后从controller的stores中获得了network。

#### 3.3.4 sandbox---getNetworkSandbox()、NewSandbox()

##### 3.3.4.1 getNetworkSandbox()

首先分析`getNetworkSandbox()`，代码位于[moby/daemon/container_operations.go#L578#L588](https://github.com/moby/moby/blob/17.05.x/daemon/container_operations.go#L578#L588)，主要代码为：

```go
func (daemon *Daemon) getNetworkSandbox(container *container.Container) libnetwork.Sandbox {
	var sb libnetwork.Sandbox
	daemon.netController.WalkSandboxes(func(s libnetwork.Sandbox) bool {
		if s.ContainerID() == container.ID {
			sb = s
			return true
		}
		return false
	})
	return sb
}
```

继续看`daemon.netController.WalkSandboxes()`，代码位于[moby/vendor/github.com/docker/libnetwork/controller.go#L1057#L1063](https://github.com/moby/moby/blob/17.05.x/vendor/github.com/docker/libnetwork/controller.go#L1057#L1063)，主要代码为：

```go
func (c *controller) WalkSandboxes(walker SandboxWalker) {
	for _, sb := range c.Sandboxes() {
		if walker(sb) {
			return
		}
	}
}
```

接着到`c.Sandboxes()`，代码位于[moby/vendor/github.com/docker/libnetwork/controller.go#L1040#L1055](https://github.com/moby/moby/blob/17.05.x/vendor/github.com/docker/libnetwork/controller.go#L1040#L1055)，主要代码为：

```go
func (c *controller) Sandboxes() []Sandbox {
	list := make([]Sandbox, 0, len(c.sandboxes))
	//sandboxes为sandboxTable类型
	for _, s := range c.sandboxes {
		// Hide stub sandboxes from libnetwork users
		if s.isStub {
			continue
		}
		list = append(list, s)
	}
	return list
}
```

所以，查找的sandbox最终从controller的sandboxes中取得。

##### 3.3.4.2 NewSandbox()

接着分析`NewSandbox()`，代码位于[moby/vendor/github.com/docker/libnetwork/controller.go#L929#L1038](https://github.com/moby/moby/blob/17.05.x/vendor/github.com/docker/libnetwork/controller.go#L929#L1038)，主要代码为：

```go
func (c *controller) NewSandbox(containerID string, options ...SandboxOption) (sBox Sandbox, err error) {
	var sb *sandbox
	c.Lock()
	//对于container已经有sandbox的处理
	for _, s := range c.sandboxes {
		if s.containerID == containerID {
			// If not a stub, then we already have a complete sandbox.
			if !s.isStub {
				sbID := s.ID()
				c.Unlock()
				return nil, types.ForbiddenErrorf("container %s is already present in sandbox %s", containerID, sbID)
			}

			// We already have a stub sandbox from the
			// store. Make use of it so that we don't lose
			// the endpoints from store but reset the
			// isStub flag.
			sb = s
			sb.isStub = false
			break
		}
	}
	c.Unlock()
	//首先创建sandbox和process options,Key的生成由option决定
	if sb == nil {
		sb = &sandbox{
			id:                 stringid.GenerateRandomID(),
			containerID:        containerID,
			endpoints:          epHeap{},
			epPriority:         map[string]int{},
			populatedEndpoints: map[string]struct{}{},
			config:             containerConfig{},
			controller:         c,
			extDNS:             []extDNSEntry{},
		}
	}
	sBox = sb
	//调用的Golang的"container/heap"包，将sandbox的endpoints初始化为heap操作
	heap.Init(&sb.endpoints)
	//处理options
	sb.processOptions(options...)
	c.Lock()
	if sb.ingress && c.ingressSandbox != nil {
		c.Unlock()
		return nil, types.ForbiddenErrorf("ingress sandbox already present")
	}
	//将新建的sandbox赋值给controller的ingressSandbox
	if sb.ingress {
		c.ingressSandbox = sb
		sb.config.hostsPath = filepath.Join(c.cfg.Daemon.DataDir, "/network/files/hosts")
		sb.config.resolvConfPath = filepath.Join(c.cfg.Daemon.DataDir, "/network/files/resolv.conf")
		sb.id = "ingress_sbox"
	}
	c.Unlock()
	//创建host file、更新parentHosts、启动DNS
	if err = sb.setupResolutionFiles(); err != nil {
		return nil, err
	}
	//使用默认sandbox
	if sb.config.useDefaultSandBox {
		c.sboxOnce.Do(func() {
			c.defOsSbox, err = osl.NewSandbox(sb.Key(), false, false)
		})
		//osSbox类型为osl.Sandbox，Package osl describes structures and interfaces which abstract os entities
		sb.osSbox = c.defOsSbox
	}
	//将新建的sandbox注册到controller的sandboxes数组中
	c.Lock()
	c.sandboxes[sb.id] = sb
	c.Unlock()
	err = sb.storeUpdate()
	return sb, nil
}
```

#### 3.3.5 endpoint---n.CreateEndpoint()

`n.CreateEndpoint()`实现位于[moby/vendor/github.com/docker/libnetwork/network.go#L889#L990](https://github.com/moby/moby/blob/17.05.x/vendor/github.com/docker/libnetwork/network.go#L889#L990)。

分析流程如图：

![](/img/study/study-6-docker-6-libnetwork-excuting-flow/libnetwork-excute-flow-container-start-CreateEndpoint.png)

主要代码为：

```go
func (n *network) CreateEndpoint(name string, options ...EndpointOption) (Endpoint, error) {
	//endpoint结构体
	ep := &endpoint{name: name, generic: make(map[string]interface{}), iface: &endpointInterface{}}
	//随机生成一个endpoint ID
	ep.id = stringid.GenerateRandomID()
	// Initialize ep.network with a possibly stale copy of n. We need this to get network from
	// store. But once we get it from store we will have the most uptodate copy possibly.
	//ep.network初始化赋值为n，是为了之后从store从获取network，从store中获取的network是最新的版本
	ep.network = n
	ep.locator = n.getController().clusterHostID()
	ep.network, err = ep.getNetworkFromStore()     //根据n，从store中获取最新的network
	n = ep.network
	//处理endpoint的option,这里的option都是一些函数(type EndpointOption func(ep *endpoint))，也就是使用option这个函数来处理endpoint
	ep.processOptions(options...)
	//遍历endpoint.iface.llAddrs,是[]*net.IPNet类型，returns the list of link-local (IPv4/IPv6) addresses assigned to the endpoint.返回的是endpoint的ip地址
	for _, llIPNet := range ep.Iface().LinkLocalAddresses() {
		//判断是否时链路本地单播地址
		if !llIPNet.IP.IsLinkLocalUnicast() {
			return nil, types.BadRequestErrorf("invalid link local IP address: %v", llIPNet.IP)
		}
	}
	/*
	netlabel.MacAddress=="com.docker.network.endpoint.macaddress",而generic为map[string]interface{},所以这里opt是个interface{},空interface（参考http://www.jb51.net/article/56812.htm），所以这个opt是任意类型,所以map存的是key为string类型，value为任意类型。这个这里的作用是，取出generic这个map的"com.docker.network.endpoint.macaddress"mac,判断mac是不是net.HardwareAddr类型的，如果是，那就将mac赋值给endpoint
	*/
	if opt, ok := ep.generic[netlabel.MacAddress]; ok {
		if mac, ok := opt.(net.HardwareAddr); ok {
			ep.iface.mac = mac
		}
	}
	//获得ipamapi.Ipam和ipamapi.Capability，Ipam是一接口，目的是可以插入/修改IPAM的数据库；Capability代表的是IPAM driver的requirements和capabilities
	ipam, cap, err := n.getController().getIPAMDriver(n.ipamType)
	//如果需要mac地址
	if cap.RequiresMACAddress {
		if ep.iface.mac == nil {
			//随机生成一个mac地址
			ep.iface.mac = netutils.GenerateRandomMAC()
		}
		if ep.ipamOptions == nil {
			ep.ipamOptions = make(map[string]string)
		}
		//将新生成的mac地址存入endpoint的ipamOptions数组中
		ep.ipamOptions[netlabel.MacAddress] = ep.iface.mac.String()
	}
	给endpoint分配ipv4和ipv6地址
	if err = ep.assignAddress(ipam, true, n.enableIPv6 && !n.postIPv6); err != nil {
		return nil, err
	}
	//向network中加入endpoint，在3.3.5.1中继续分析
	if err = n.addEndpoint(ep); err != nil {
		return nil, err
	}
	//？？再分配一次地址，不知道为什么要两次
	if err = ep.assignAddress(ipam, false, n.enableIPv6 && n.postIPv6); err != nil {
		return nil, err
	}
	//将该network的endpoint信息存入controller的store中
	if err = n.getController().updateToStore(ep); err != nil {
		return nil, err
	}
	// Watch for service records
	n.getController().watchSvcRecord(ep)
	// Increment endpoint count to indicate completion of endpoint addition
	if err = n.getEpCnt().IncEndpointCnt(); err != nil {
		return nil, err
	}
	return ep, nil
}
```

##### 3.3.5.1 n.addEndpoint(ep)

`n.addEndpoint(ep)`的实现位于[moby/vendor/github.com/docker/libnetwork/network.go#L874#L887](https://github.com/moby/moby/blob/17.05.x/vendor/github.com/docker/libnetwork/network.go#L874#L887)，主要代码为：

```go
func (n *network) addEndpoint(ep *endpoint) error {
	//获得网络驱动,3.3.5.2分析
	d, err := n.driver(true)
	//调用网络驱动创建endpoint，有bridge、host等,3.3.5.3分析
	err = d.CreateEndpoint(n.id, ep.id, ep.Interface(), ep.generic)
	return nil
}
```

##### 3.3.5.2 n.driver(true)

`n.driver(true)`的实现位于[moby/vendor/github.com/docker/libnetwork/network.go#L760#L781](https://github.com/moby/moby/blob/17.05.x/vendor/github.com/docker/libnetwork/network.go#L760#L781)，主要代码为：

```go
func (n *network) driver(load bool) (driverapi.Driver, error) {
	//根据网络类型获取网络驱动，如果没有，则看load是否为true，是true的话，就载入该网络类型的驱动
	d, cap, err := n.resolveDriver(n.networkType, load)
	c := n.getController()
	//returns true if Cluster is participating as a worker/agent，查看该节点是worker还是agent.
	isAgent := c.isAgent()
	n.Lock()
	// If load is not required, driver, cap and err may all be nil
	if cap != nil {
		n.scope = cap.DataScope
	}
	if isAgent || n.dynamic {
		// If we are running in agent mode then all networks
		// in libnetwork are local scope regardless of the
		// backing driver.
		n.scope = datastore.LocalScope
	}
	n.Unlock()
	return d, nil
}
```

##### 3.3.5.3 d.CreateEndpoint()

现在看d.CreateEndpoint(n.id, ep.id, ep.Interface(), ep.generic)调用，d即driver的种类有：bridge、host、ipvlan、macvlan、overlay、remote六种，每种驱动都有`CreateEndpoint()`的实现，这里我们先来看看brdige驱动的`CreateEndpoint()`，实现位于[moby/vendor/github.com/docker/libnetwork/drivers/bridge/bridge.go#L902#L1081](https://github.com/moby/moby/blob/17.05.x/vendor/github.com/docker/libnetwork/drivers/bridge/bridge.go#L902#L1081)，主要代码为：

```go
func (d *driver) CreateEndpoint(nid, eid string, ifInfo driverapi.InterfaceInfo, epOptions map[string]interface{}) error {
	//InitOSContext initializes OS context while configuring network resources
	defer osl.InitOSContext()()
	// Get the network handler and make sure it exists
	d.Lock()
	//根据network id获得container的network，类型为bridgeNetwork
	n, ok := d.networks[nid]
	dconfig := d.config
	d.Unlock()
	
	// Sanity check,检查network的连通性
	n.Lock()
	if n.id != nid {
		n.Unlock()
		return InvalidNetworkIDError(nid)
	}
	n.Unlock()
	// Check if endpoint id is good and retrieve correspondent endpoint.类型为bridgeEndpoint
	ep, err := n.getEndpoint(eid)
	// Endpoint with that id exists either on desired or other sandbox
	//这里有一些不理解，ep!=nil，说明id是我们所有需要的或存在于别的sandbox中？
	if ep != nil {
		return driverapi.ErrEndpointExists(eid)
	}
	// Try to convert the options to endpoint configuration
	epConfig, err := parseEndpointOptions(epOptions)
	// Create and add the endpoint
	n.Lock()
	endpoint := &bridgeEndpoint{id: eid, nid: nid, config: epConfig}
	n.endpoints[eid] = endpoint
	n.Unlock()
	// Generate a name for what will be the host side pipe interface
	//生成在host端的pipe的名字
	hostIfName, err := netutils.GenerateIfaceName(d.nlh, vethPrefix, vethLen)
	// Generate a name for what will be the sandbox side pipe interface
	//生成在sandbox端的pipe的名字
	containerIfName, err := netutils.GenerateIfaceName(d.nlh, vethPrefix, vethLen)
	// Generate and add the interface pipe host <-> sandbox
	//生成veth pair
	veth := &netlink.Veth{
		LinkAttrs: netlink.LinkAttrs{Name: hostIfName, TxQLen: 0},
		PeerName:  containerIfName}
	//添加veth pair
	/*
	LinkAdd adds a new link device. The type and features of the device are taken fromt the parameters in the link object. Equivalent to: `ip link add $link`
	*/
	if err = d.nlh.LinkAdd(veth); err != nil {
		return types.InternalErrorf("failed to add the host (%s) <=> sandbox (%s) pair interfaces: %v", hostIfName, containerIfName, err)
	}
	// Get the host side pipe interface handler
	host, err := d.nlh.LinkByName(hostIfName)
	// Get the sandbox side pipe interface handler
	sbox, err := d.nlh.LinkByName(containerIfName)
	//获得bridgeNetwork的配置
	n.Lock()
	config := n.config
	n.Unlock()
	// Add bridge inherited attributes to pipe interfaces
	if config.Mtu != 0 {
		//设置host端veth的MTU
		err = d.nlh.LinkSetMTU(host, config.Mtu)
		//设置sandbox端vethpair的MTU
		err = d.nlh.LinkSetMTU(sbox, config.Mtu)
	}
	// Attach host side pipe interface into the bridge，之后分析
	if err = addToBridge(d.nlh, hostIfName, config.BridgeName); 
	//UserlandProxy,每增加一个端口映射，就会增加一个docker-proxy进程，实现宿主机上0.0.0.0地址上对容器的访问代理。
	//参考http://www.dataguru.cn/thread-544489-1-1.html
	//这里是对不开启UserlandProxy的情况做处理
	if !dconfig.EnableUserlandProxy {
		/*
			什么是Hairpin？参考http://www.bubuko.com/infodetail-1994270.html
			 这是一个网络虚拟化技术中常提到的概念，也即交换机端口的VEPA模式。这种技术借助物理交换机解决了虚拟机间流量转发问题。很显然，这种情况下，源和目标都在一个方向，所以就是从哪里进从哪里出的模式。
		*/
		err = setHairpinMode(d.nlh, host, true)
		if err != nil {
			return err
		}
	}
	// Store the sandbox side pipe interface parameters
	endpoint.srcName = containerIfName
	endpoint.macAddress = ifInfo.MacAddress()
	endpoint.addr = ifInfo.Address()
	endpoint.addrv6 = ifInfo.AddressIPv6()
	// Set the sbox's MAC if not provided. If specified, use the one configured by user, otherwise generate one based on IP.
	if endpoint.macAddress == nil {
		endpoint.macAddress = electMacAddress(epConfig, endpoint.addr.IP)
		if err = ifInfo.SetMacAddress(endpoint.macAddress)
	// Up the host interface after finishing all netlink configuration
	if err = d.nlh.LinkSetUp(host);
	//对启用了ipv6的情况进行处理
	if endpoint.addrv6 == nil && config.EnableIPv6 {
		var ip6 net.IP
		network := n.bridge.bridgeIPv6
		if config.AddressIPv6 != nil {
			network = config.AddressIPv6
		}

		ones, _ := network.Mask.Size()
		if ones > 80 {
			err = types.ForbiddenErrorf("Cannot self generate an IPv6 address on network %v: At least 48 host bits are needed.", network)
			return err
		}
		//生成ipv6地址
		ip6 = make(net.IP, len(network.IP))
		copy(ip6, network.IP)
		for i, h := range endpoint.macAddress {
			ip6[i+10] = h
		}

		endpoint.addrv6 = &net.IPNet{IP: ip6, Mask: network.Mask}
		if err = ifInfo.SetIPAddress(endpoint.addrv6); err != nil {
			return err
		}
	}
	//endpoint信息更新到store中
	if err = d.storeUpdate(endpoint);
	return nil
}	
```

这里再看看`addToBridge(d.nlh, hostIfName, config.BridgeName)`，实现位于[moby/vendor/github.com/docker/libnetwork/drivers/bridge/bridge.go#L848#L869](https://github.com/moby/moby/blob/17.05.x/vendor/github.com/docker/libnetwork/drivers/bridge/bridge.go#L848#L869)，主要代码为：

```go
func addToBridge(nlh *netlink.Handle, ifaceName, bridgeName string) error {
	//finds a link by name and returns a pointer to the object.
	//获得host端veth
	link, err := nlh.LinkByName(ifaceName)
	// LinkSetMaster sets the master of the link device.
	// Equivalent to: `ip link set $link master $master`
	下面详细再看看
	nlh.LinkSetMaster(link,&netlink.Bridge{LinkAttrs: netlink.LinkAttrs{Name: bridgeName}})
	return nil
}
```

接着到`nlh.LinkSetMaster(link,&netlink.Bridge{LinkAttrs: netlink.LinkAttrs{Name: bridgeName}})`的调用，它的实现位于[moby/vendor/github.com/vishvananda/netlink/link_linux.go#L380#L391](https://github.com/moby/moby/blob/17.05.x/vendor/github.com/vishvananda/netlink/link_linux.go#L380#L391)，主要代码为：

```go
func (h *Handle) LinkSetMaster(link Link, master *Bridge) error {
	index := 0
	if master != nil {
		//返回一个LinkAttrs结构体，LinkAttrs represents data shared by most link types，这个结构体包含了MTU、TxQLen等一些属性
		masterBase := master.Attrs()
		//获取正确的link index
		h.ensureIndex(masterBase)
		index = masterBase.Index
	}
	if index <= 0 {
		return fmt.Errorf("Device does not exist")
	}
	// LinkSetMasterByIndex sets the master of the link device.
	// Equivalent to: `ip link set $link master $master`
	//根据index设置，下面再详细看
	return h.LinkSetMasterByIndex(link, index)
}
```

接着到`h.LinkSetMasterByIndex(link, index)`的调用，它的实现位于[moby/vendor/github.com/vishvananda/netlink/link_linux.go#L413#L430](https://github.com/moby/moby/blob/17.05.x/vendor/github.com/vishvananda/netlink/link_linux.go#L413#L430)，主要代码为：

```go
func (h *Handle) LinkSetMasterByIndex(link Link, masterIndex int) error {
	//这里的link是./vishvananda/netlink/link.go中的link，返回的也是一个LinkAttrs结构体，这里返回的是host的veth的
	base := link.Attrs()
	h.ensureIndex(base)
	//生成一个NetlinkRequest结构体
	req := h.newNetlinkRequest(syscall.RTM_SETLINK, syscall.NLM_F_ACK)
	// Create an IfInfomsg with family specified
	msg := nl.NewIfInfomsg(syscall.AF_UNSPEC)
	msg.Index = int32(base.Index)
	//req添加Data
	req.AddData(msg)

	b := make([]byte, 4)
	native.PutUint32(b, uint32(masterIndex))
	// Create a new Extended RtAttr object
	data := nl.NewRtAttr(syscall.IFLA_MASTER, b)
	req.AddData(data)
	// Execute the request against a the given sockType.
	// Returns a list of netlink messages in serialized format, optionally filtered by resType.
	//通过给定的信息执行NetlinkRequest
	_, err := req.Execute(syscall.NETLINK_ROUTE, 0)
	return err
}
```

目前就先看到这里，再接下去的地方好些太底层了。

#### 3.3.6 ep.Join(sb, joinOptions...)

接着是`ep.Join(sb, joinOptions...)`，分析流程如图：

![](/img/study/study-6-docker-6-libnetwork-excuting-flow/libnetwork-excute-flow-container-start-Join.png)

该函数的实现位于[moby/vendor/github.com/docker/libnetwork/endpoint.go#L414#L428](https://github.com/moby/moby/blob/17.05.x/vendor/github.com/docker/libnetwork/endpoint.go#L414#L428)，主要代码为：

```go
func (ep *endpoint) Join(sbox Sandbox, options ...EndpointOption) error {
	//将Sandbox一般类型转为sandbox类型，参考http://www.jb51.net/article/119585.htm-类型断言
	sb, ok := sbox.(*sandbox)
	//joinLeaveStart waits to ensure there are no joins or leaves in progress and  marks this join/leave in progress without race
	//确保sandbox没有竞争
	sb.joinLeaveStart()
	defer sb.joinLeaveEnd()
	//下面继续分析
	return ep.sbJoin(sb, options...)
}
```

这里来到`ep.sbJoin(sb, options...)`的实现，位于[moby/vendor/github.com/docker/libnetwork/endpoint.go#L430#L580](https://github.com/moby/moby/blob/17.05.x/vendor/github.com/docker/libnetwork/endpoint.go#L430#L580)，主要代码为：

接下来分析
```go
func (ep *endpoint) sbJoin(sb *sandbox, options ...EndpointOption) (err error) {
	//从store中获取endpoint对应的network
	n, err := ep.getNetworkFromStore()
	//从store中获取network对应的endpoint
	ep, err = n.getEndpointFromStore(ep.ID())
	//给endpoint赋值一些基本信息
	ep.Lock()
	if ep.sandboxID != "" {
		ep.Unlock()
		return types.ForbiddenErrorf("another container is attached to the same network endpoint")
	}
	ep.network = n
	ep.sandboxID = sb.ID()
	ep.joinInfo = &endpointJoinInfo{}
	epid := ep.id
	ep.Unlock()
	//network ID
	nid := n.ID()
	//执行endpoint的options里的一系列函数
	ep.processOptions(options...)
	//获取网络驱动
	d, err := n.driver(true)
	//调用网络驱动的Join函数，endpoint加入到sandbox中，将在3.3.6.1中分析
	err = d.Join(nid, epid, sb.Key(), ep, sb.Labels())
	// Watch for service records
	if !n.getController().isAgent() {
		n.getController().watchSvcRecord(ep)
	}
	// 如果!n.ingress && n.Name() != libnGWNetwork，则进入if
	if doUpdateHostsFile(n, sb) {
		address := ""
		if ip := ep.getFirstInterfaceAddress(); ip != nil {
			address = ip.String()
		}
		if err = sb.updateHostsFile(address); err != nil {
			return err
		}
	}
	//更新DNS
	if err = sb.updateDNS(n.enableIPv6); err != nil {
		return err
	}
	// Current endpoint providing external connectivity for the sandbox
	// Returns the endpoint which is providing external connectivity to the sandbox
	extEp := sb.getGatewayEndpoint()
	
	sb.Lock()
	//将endpoint压入sandbox的endpoints heap中
	heap.Push(&sb.endpoints, ep)
	sb.Unlock()
	//Sandbox每次通过一个Endpoint去Join一个Network都会去发布一把这个ep，参考http://www.jianshu.com/p/4433f4c70cf0
	sb.populateNetworkResources(ep)
	//将endpoint信息更新到store中
	n.getController().updateToStore(ep)
	//将driver信息添加到Cluster中
	ep.addDriverInfoToCluster()
	/*
	needDefaultGW():Evaluate whether the sandbox requires a default gateway based on the endpoints to which it is connected. It does not account for the default gateway network endpoint.
	getEndpointInGWNetwork():如果GWNetwork为空
	*/
	if sb.needDefaultGW() && sb.getEndpointInGWNetwork() == nil {
		/*
  		  libnetwork creates a bridge network "docker_gw_bridge" for providing
 		  default gateway for the containers if none of the container's endpoints
 		  have GW set by the driver. ICC is set to false for the GW_bridge network.

  		  If a driver can't provide external connectivity it can choose to not set
   		  the GW IP for the endpoint.

  		 endpoint on the GW_bridge network is managed dynamically by libnetwork.
   		 ie:
   			- its created when an endpoint without GW joins the container
   			- its deleted when an endpoint with GW joins the container
		*/
		return sb.setupDefaultGW()
	}
	//如果GatewayEndpoint改变了
	moveExtConn := sb.getGatewayEndpoint() != extEp
	if moveExtConn {
		if extEp != nil {
			//撤销外部连接endpoint
			logrus.Debugf("Revoking external connectivity on endpoint %s (%s)", extEp.Name(), extEp.ID())
			//获得extEp对应的network
			extN, err := extEp.getNetworkFromStore()
			//获得expEp对应network对应的driver
			extD, err := extN.driver(true)
			//撤销endpoint的外部连接
			extD.RevokeExternalConnectivity(extEp.network.ID(), extEp.ID())
		}
		//network不是内部的	
		if !n.internal {
			logrus.Debugf("Programming external connectivity on endpoint %s (%s)", ep.Name(), ep.ID())
			d.ProgramExternalConnectivity(n.ID(), ep.ID(), sb.Labels())
		}
	}
	//sandbox不需要默认Gateway
	if !sb.needDefaultGW(){
		// If present, detach and remove the endpoint connecting the sandbox to the default gw network.
		sb.clearDefaultGW()
	}
	return nil
}
```

##### 3.3.6.1 d.Join()

`d.Join(nid, epid, sb.Key(), ep, sb.Labels())`分析bridge驱动下的实现，位于[moby/vendor/github.com/docker/libnetwork/drivers//bridge/bridge.go#L1204#L1247](https://github.com/moby/moby/blob/17.05.x/vendor/github.com/docker/libnetwork/drivers//bridge/bridge.go#L1204#L1247)，主要代码为：

```go
func (d *driver) Join(nid, eid string, sboxKey string, jinfo driverapi.JoinInfo, options map[string]interface{}) error {
	// InitOSContext initializes OS context while configuring network resources
	defer osl.InitOSContext()()
	network, err := d.getNetwork(nid)
	endpoint, err := network.getEndpoint(eid)
	//从options中获得containerConfig
	endpoint.containerConfig, err = parseContainerOptions(options)
	// InterfaceName returns an InterfaceNameInfo go interface to facilitate setting the names for the interface.
	// 返回一个InterfaceNameInfo的接口，用于名字的设置,返回的是ep.iface,类型endpointInterface
	iNames := jinfo.InterfaceName()
	// defaultContainerVethPrefix = "eth" 
	containerVethPrefix := defaultContainerVethPrefix
	if network.config.ContainerIfacePrefix != "" {
		containerVethPrefix = network.config.ContainerIfacePrefix
	}
	//设置endpoint srcName和containerVethPrefix
	iNames.SetNames(endpoint.srcName, containerVethPrefix)
	// SetGateway sets the default IPv4 gateway when a container joins the endpoint.
	err = jinfo.SetGateway(network.bridge.gatewayIPv4)
	// SetGatewayIPv6 sets the default IPv6 gateway when a container joins the endpoint.
	err = jinfo.SetGatewayIPv6(network.bridge.gatewayIPv6)
	return nil
}
```

## 结语

本文分析了daemon对container start的处理做了一个简单分析，很多细节部分没有得到很好的分析，比如网络的具体创建、从daemon到containerd的传递，这些内容将在后续的博客中分析。

## 参考

* *[Docker 网络部分执行流分析（libnetwork源码解读）](http://blog.csdn.net/gao514916467/article/details/51242299)*
* *[从docker daemon的角度，添加了userland-proxy的起停开关](http://www.dataguru.cn/thread-544489-1-1.html)*
* *[Linux Bridge的IP NAT细节探析-填补又一坑的过程](http://www.bubuko.com/infodetail-1994270.html)*
* *[浅谈Go语言中的结构体struct & 接口Interface & 反射](http://www.jb51.net/article/119585.htm)*
* *[Docker Embedded DNS](http://www.jianshu.com/p/4433f4c70cf0)*
