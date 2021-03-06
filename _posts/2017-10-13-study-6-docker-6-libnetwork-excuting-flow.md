---
layout:     post
title:      "「六」DOCKER源码分析6 网络部分执行流分析（libnetwork源码解读）"
date:       2017-10-13 11:00:00 
author:     "guanjunjian"
categories: DOCKER源码分析
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




## 2. libnetwork在docker daemon初始化时的工作

### 2.1 libnetwork的工作

在daemon初始化时，libnetwork的工作主要有两个：

* 在Docker daemon启动的时候，daemon会创建一个bridge驱动所对应的netcontroller，这个controller可以在后面的流程里面对network和sandbox等进行创建;
* 接下来daemon通过调用controller.NewNetwork（）来创建指定类型（bridge类型）和指定名称（即docker0）的network。Libnetwork在接收到创建命令后会使用系统linux的系统调用，创建一个名为docker0的网桥。我们可以在Docker daemon启动后，使用ifconfig命令看到这个名为docker0的网桥。

### 2.2 流程图

流程图如下（图中main函数位于[moby/cmd/dockerd/docker.go#100#122](https://github.com/moby/moby/blob/17.05.x/cmd/dockerd/docker.go#L100#L122)）：

![](/img/study/study-6-docker-6-libnetwork-excuting-flow/libnetwork-excute-flow-daemon-init.png)

### 2.3 代码分析---initNetworkController()

**initNetworkController()**

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

这部分的工作有：
* 1.调用libnetwork的New函数创建NetworkController
* 2.向controller注册none、host、bridge三个默认network

下面在2.3.1分析生成NetworkController的`libnetwork.New()`,在2.3.2分析生成Network的`controller.NewNetwork()`以及在2.3.3分析`initBridgeDriver()`。

#### 2.3.1 NetworkController---libnetwork.New()

**libnetwork.New()**

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

这部分的工作有：
* 1.生成并初始化controller结构体
* 2.初始化stores
* 3.生成drvregistry,用于存储网络驱动和IPAM驱动
* 4.将bridge、host、macvlan、null、remote、overlay等驱动初始化函数注册到drvregistry中
* 5.初始化IPAM驱动注册到drvregistry中
* 6.populate该controller中的所有network
* 7./run/docker/libnetwork/container ID.sock开始监听

下面分别在2.3.1.1分析`4.`中提到的bridge驱动初始化函数，在2.3.1.2中分析`7.`中提到的.sock文件的监听。

**2.3.1.1 libnetwork.New()--->bridge驱动的初始化**

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


**2.3.1.2 libnetwork.New()--->c.startExternalKeyListener()**

`c.startExternalKeyListener()`实现位于[moby/vendor/github.com/docker/libnetwork/sandbox_externalkey_unix.go#L105#L124](https://github.com/moby/moby/blob/17.05.x/vendor/github.com/docker/libnetwork/sandbox_externalkey_unix.go#L105#L124)。对于这部分的代码，分析如下：

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


#### 2.3.2 Network---controller.NewNetwork()

**controller.NewNetwork()**

`controller.NewNetwork()`的实现位于[moby/vendor/github.com/docker/libnetwork/controller.go#L665#L773](https://github.com/moby/moby/blob/17.05.x/vendor/github.com/docker/libnetwork/controller.go#L665#L773)。以

```go
controller.NewNetwork("bridge", "bridge", "",
		libnetwork.NetworkOptionEnableIPv6(config.BridgeConfig.EnableIPv6),
		libnetwork.NetworkOptionDriverOpts(netOption),
		libnetwork.NetworkOptionIpam("default", "", v4Conf, v6Conf, nil),
		libnetwork.NetworkOptionDeferIPv6Alloc(deferIPv6Alloc))
```

为例，代码分析如下：

```go
func (c *controller) NewNetwork(networkType, name string, id string, options ...NetworkOption) (Network, error) {
	//对于例子中，id=="",这里是查看对应id的network是否已经存在，如果已经存在则返回错误
	if id != "" {
		c.networkLocker.Lock(id)
		defer c.networkLocker.Unlock(id)

		if _, err := c.NetworkByID(id); err == nil {
			return nil, NetworkNameError(id)
		}
	}
	//检验network name
	if !config.IsValidName(name) {
		return nil, ErrInvalidName(name)
	}
	//随机生成一个network id
	if id == "" {
		id = stringid.GenerateRandomID()
	}
	// 返回的是 "default"，是default ipam driver的名字
	defaultIpam := defaultIpamForNetworkType(networkType)
	//创建network结构体
	network := &network{
		name:        name,
		networkType: networkType,
		generic:     map[string]interface{}{netlabel.GenericData: make(map[string]string)},
		ipamType:    defaultIpam,
		id:          id,
		created:     time.Now(),
		ctrlr:       c,
		persist:     true,
		drvOnce:     &sync.Once{},
	}
	//network的options挨个进行调用
	network.processOptions(options...)
	//根据网络类型，获得驱动以及驱动的capability，true表示如果没在找到这种类型的network的驱动，则进行装载
	_, cap, err := network.resolveDriver(networkType, true)
	//如果network的ingress为true，cap的DataScope不为"global"
	if network.ingress && cap.DataScope != datastore.GlobalScope {
		//返回错误，Ingress network只能是global scope network
		return nil, types.ForbiddenErrorf("Ingress network can only be global scope network")
	}
	// cap的DataScope为"global"，controller是manager和agent，network的dynamic为false
	if cap.DataScope == datastore.GlobalScope && !c.isDistributedControl() && !network.dynamic {
		if c.isManager() {
			// For non-distributed controlled environment, globalscoped non-dynamic networks are redirected to Manager
			return nil, ManagerRedirectError(name)
		}
		//不能在worker节点创建multi-host网络
		return nil, types.ForbiddenErrorf("Cannot create a multi-host network from a worker node. Please create the network from a manager node.")
	}
	//在分配各种资源前，确保这种类型的网络驱动是可用的
	if _, err := network.driver(true); err != nil {
		return nil, err
	}
	//分配ipam
	network.ipamAllocate()
	//创建network，下面继续分析
	c.addNetwork(network)
	/*
	First store the endpoint count, then the network.
	To avoid to end up with a datastore containing a network and not an epCnt,in case of an ungraceful shutdown during this function call.
	*/
	epCnt := &endpointCnt{n: network}
	if err = c.updateToStore(epCnt); err != nil {
		return nil, err
	}
	//network的endpoint count
	network.epCnt = epCnt
	//将network更新存储到store中
	c.updateToStore(network)
	//join network into agent cluster
	joinCluster(network)
	if !c.isDistributedControl() {
		c.Lock()
		/*
		In the filter table FORWARD chain first rule should be to jump to INGRESS-CHAIN.
		This chain has the rules to allow access to the published ports for swarm tasks from local bridge networks and docker_gwbridge (ie:taks on other swarm netwroks)
		*/
		arrangeIngressFilterRule()
		c.Unlock()
	}
	return network, nil
}
```

这部分的作用：
* 1.对network id、network name进行有效性验证
* 2.创建并初始化network结构体，对network结构体进行options处理
* 3.解析networkTpye，并获取该networkTpye的capability,并根据capability进行一些错误检查。
* 4.确保network驱动可用，如果不可用则装载
* 5.为network分配ipam
* 6.将network添加到controller中
* 7.设置network的endpoint计数

下面在2.3.2.1分析`6.`中提到的`c.addNetwork(network)`，这其中涉及到了调用driver对network进行创建。

**2.3.2.1 controller.NewNetwork()--->c.addNetwork(network)**

`c.addNetwork(network)`的实现位于[moby/vendor/github.com/docker/libnetwork/controller.go#L580#L864](https://github.com/moby/moby/blob/17.05.x/vendor/github.com/docker/libnetwork/controller.go#L580#L864)，分析如下:

```go
func (c *controller) addNetwork(n *network) error {
	//获得对应network的driver，如果不存在则载入
	d, err := n.driver(true)
	// Create the network，下面接着分析
	d.CreateNetwork(n.id, n.generic, n, n.getIPData(4), n.getIPData(6))
	//在linux中是个空函数，Stub implementations for DNS related functions
	n.startResolver()
	return nil
}
```

**controller.NewNetwork()--->c.addNetwork(network)--->d.CreateNetwork()**

`d.CreateNetwork()`是以bridge driver为例的，所以它的实现位于[moby/vendor/github.com/docker/libnetwork/drivers/bridge/bridge.go#L586#L614](https://github.com/moby/moby/blob/17.05.x/vendor/github.com/docker/libnetwork/drivers/bridge/bridge.go#L586#L614)，分析如下:

```go
func (d *driver) CreateNetwork(id string, option map[string]interface{}, nInfo driverapi.NetworkInfo, ipV4Data, ipV6Data []driverapi.IPAMData) error {
	if len(ipV4Data) == 0 || ipV4Data[0].Pool.String() == "0.0.0.0/0" {
		return types.BadRequestErrorf("ipv4 pool is empty")
	}
	// Sanity checks
	d.Lock()
	if _, ok := d.networks[id]; ok {
		d.Unlock()
		//这个network id已经存在
		return types.ForbiddenErrorf("network %s exists", id)
	}
	d.Unlock()
	// Parse and validate the config. It should not be conflict with existing networks' config,config为networkConfiguration类型
	config, err := parseNetworkOptions(id, option)
	//对ip地址的处理
	config.processIPAM(id, ipV4Data, ipV6Data)
	//下面接着分析
	d.createNetwork(config)
	//更新config到store中
	return d.storeUpdate(config)
}
```

**controller.NewNetwork()--->c.addNetwork(network)--->d.CreateNetwork()--->d.createNetwork(config)**

`d.createNetwork(config)`的实现位于[moby/vendor/github.com/docker/libnetwork/drivers/bridge/bridge.go#L616#L770](https://github.com/moby/moby/blob/17.05.x/vendor/github.com/docker/libnetwork/drivers/bridge/bridge.go#L616#L770)，分析如下:

```go
func (d *driver) createNetwork(config *networkConfiguration) error {
	//initializes OS context while configuring network resources
	defer osl.InitOSContext()()
	//把d.networks转成slice
	networkList := d.getNetworks()
	
	for i, nw := range networkList {
		nw.Lock()
		nwConfig := nw.config
		nw.Unlock()
		//Conflicts check if two NetworkConfiguration objects overlap,查看新建的network config是否与之前的network冲突
		if err := nwConfig.Conflicts(config); err != nil {
			//对于新建network config冲突的处理
			//如果是使用的defaultBridge
			if config.DefaultBridge {
				// We encountered and identified a stale default network
				// We must delete it as libnetwork is the source of thruth
				// The default network being created must be the only one
				// This can happen only from docker 1.12 on ward
				logrus.Infof("Removing stale default bridge network %s (%s)", nwConfig.ID, nwConfig.BridgeName)
				if err := d.DeleteNetwork(nwConfig.ID); err != nil {
					logrus.Warnf("Failed to remove stale default network: %s (%s): %v. Will remove from store.", nwConfig.ID, nwConfig.BridgeName, err)
					d.storeDelete(nwConfig)
				}
				networkList = append(networkList[:i], networkList[i+1:]...)
			} else {
				return types.ForbiddenErrorf("cannot create network %s (%s): conflicts with network %s (%s): %s",
					config.ID, config.BridgeName, nwConfig.ID, nwConfig.BridgeName, err.Error())
			}
		}
	}
	// Create and set network handler in driver
	network := &bridgeNetwork{
		id:         config.ID,
		endpoints:  make(map[string]*bridgeEndpoint),
		config:     config,
		portMapper: portmapper.New(d.config.UserlandProxyPath),
		driver:     d,
	}
	d.Lock()
	//新建的network存入driver
	d.networks[config.ID] = network
	d.Unlock()
	// Initialize handle when needed
	d.Lock()
	if d.nlh == nil {
		// NlHandle returns the netlink handler,*netlink.Handle
		d.nlh = ns.NlHandle()
	}
	d.Unlock()
	/*
	Create or retrieve the bridge L3 interface
	newInterface creates a new bridge interface structure. It attempts to find an already existing device identified by the configuration BridgeName field, or the default bridge name when unspecified, but doesn't attempt to create one when missing
	下面接着分析
	*/
	bridgeIface, err := newInterface(d.nlh, config)
	network.bridge = bridgeIface
	/*
	Verify the network configuration does not conflict with previously installed networks. This step is needed now because driver might have now set the bridge name on this config struct. And because we need to check for possible address conflicts, so we need to check against operationa lnetworks.
	*/
	config.conflictsWithNetworks(config.ID, networkList)

	setupNetworkIsolationRules := func(config *networkConfiguration, i *bridgeInterface) error {
		/*
		isolateNetwork():Install/Removes the iptables rules needed to isolate this network from each of the other networks.
		true是install，false是removes
		*/
		if err := network.isolateNetwork(networkList, true); err != nil {
			if err := network.isolateNetwork(networkList, false); err != nil {
				logrus.Warnf("Failed on removing the inter-network iptables rules on cleanup: %v", err)
			}
			return err
		}
		//注册Iptables清理函数
		network.registerIptCleanFunc(func() error {
			nwList := d.getNetworks()
			//清理driver中所有的network的iptables
			return network.isolateNetwork(nwList, false)
		})
		return nil
	}
	// Prepare the bridge setup configuration， bridgeSetup结构体
	bridgeSetup := newBridgeSetup(config, bridgeIface)
	/*
	If the bridge interface doesn't exist, we need to start the setup steps by creating a new device and assigning it an IPv4 address.
	*/
	bridgeAlreadyExists := bridgeIface.exists()
	if !bridgeAlreadyExists {
		// SetupDevice create a new bridge interface，创建一个link并分配mac地址，下面接着分析
		bridgeSetup.queueStep(setupDevice)
	}
	// Even if a bridge exists try to setup IPv4.给bridge分配ipv4和gateway ipv4地址
	bridgeSetup.queueStep(setupBridgeIPv4)
	//是否允许ipv6转发
	enableIPv6Forwarding := d.config.EnableIPForwarding && config.AddressIPv6 != nil
	// Conditionally queue setup steps depending on configuration values.
	//依次将下面的setupstep加入到bridgeSetup.queueStep中
	for _, step := range []struct {
		Condition bool
		Fn        setupStep
	}{
		// Enable IPv6 on the bridge if required. We do this even for a
		// previously  existing bridge, as it may be here from a previous
		// installation where IPv6 wasn't supported yet and needs to be
		// assigned an IPv6 link-local address.
		{config.EnableIPv6, setupBridgeIPv6},

		// We ensure that the bridge has the expectedIPv4 and IPv6 addresses in
		// the case of a previously existing device.
		{bridgeAlreadyExists, setupVerifyAndReconcile},

		// Enable IPv6 Forwarding
		{enableIPv6Forwarding, setupIPv6Forwarding},

		// Setup Loopback Adresses Routing
		{!d.config.EnableUserlandProxy, setupLoopbackAdressesRouting},

		// Setup IPTables.
		{d.config.EnableIPTables, network.setupIPTables},

		//We want to track firewalld configuration so that
		//if it is started/reloaded, the rules can be applied correctly
		{d.config.EnableIPTables, network.setupFirewalld},

		// Setup DefaultGatewayIPv4
		{config.DefaultGatewayIPv4 != nil, setupGatewayIPv4},

		// Setup DefaultGatewayIPv6
		{config.DefaultGatewayIPv6 != nil, setupGatewayIPv6},

		// Add inter-network communication rules.
		{d.config.EnableIPTables, setupNetworkIsolationRules},

		//Configure bridge networking filtering if ICC is off and IP tables are enabled
		{!config.EnableICC && d.config.EnableIPTables, setupBridgeNetFiltering},
	} {
		if step.Condition {
			bridgeSetup.queueStep(step.Fn)
		}
	}
	// Apply the prepared list of steps, and abort at the first error. 
	//SetupDeviceUp ups the given bridge interface.
	bridgeSetup.queueStep(setupDeviceUp)
	//运行上面所有的setupsteps
	bridgeSetup.apply()
	return nil
}
```
这部分的作用是：
* 1.根据所要新建的network的config，验证是否与该driver的其他已经存在的network的config冲突
* 2.新建bridgeNetwork结构体并存储到driver的networks数组中
* 3.为driver的nlh赋值ns.NlHandle()，类型为*netlink.Handle
* 4.创建bridgeIface，并赋值给network结构体
* 5.再次冲突性验证
* 6.设置network的隔离
* 7.设置network的setup config，为network的启动做准备，其中包括setupDevice,它创建一个link

下面接着分析`4.`中提到的`newInterface(d.nlh, config)`和`7.`中提到的`bridgeSetup.queueStep(setupDevice)`。

**controller.NewNetwork()--->c.addNetwork(network)--->d.CreateNetwork()--->d.createNetwork(config)--->newInterface(d.nlh, config)**

`newInterface(d.nlh, config)`的实现位于[moby/vendor/github.com/docker/libnetwork/drivers/bridge/interface.go#L31#L48](https://github.com/moby/moby/blob/17.05.x/vendor/github.com/docker/libnetwork/drivers/bridge/interface.go#L31#L48)，分析如下:

```go
func newInterface(nlh *netlink.Handle, config *networkConfiguration) (*bridgeInterface, error) {
	// create interface structure
	i := &bridgeInterface{nlh: nlh}
	// Initialize the bridge name to the default if unspecified.
	if config.BridgeName == "" {
		config.BridgeName = DefaultBridgeName
	}
	// Attempt to find an existing bridge named with the specified name.
	i.Link, err = nlh.LinkByName(config.BridgeName)
	if err != nil {
		logrus.Debugf("Did not find any interface with name %s: %v", config.BridgeName, err)
	}
	//将link转为netlink.Bridge类型 
	else if _, ok := i.Link.(*netlink.Bridge); !ok {
		return nil, fmt.Errorf("existing interface %s is not a bridge", i.Link.Attrs().Name)
	}
	return i, nil
}
```

**controller.NewNetwork()--->c.addNetwork(network)--->d.CreateNetwork()--->d.createNetwork(config)--->bridgeSetup.queueStep(setupDevice)**

`bridgeSetup.queueStep(setupDevice)`中的setupDevice的作用是创建一个新的bridge interface，它的实现位于[moby/vendor/github.com/docker/libnetwork/drivers/bridge/setup_device.go#L13#L51](https://github.com/moby/moby/blob/17.05.x/vendor/github.com/docker/libnetwork/drivers/bridge/setup_device.go#L13#L51)，分析如下:

```go
func setupDevice(config *networkConfiguration, i *bridgeInterface) error {
	var setMac bool
	// We only attempt to create the bridge when the requested device name is the default one.
	if config.BridgeName != DefaultBridgeName && config.DefaultBridge {
		return NonDefaultBridgeExistError(config.BridgeName)
	}
	// Set the bridgeInterface netlink.Bridge.
	i.Link = &netlink.Bridge{
		LinkAttrs: netlink.LinkAttrs{
			Name: config.BridgeName,
		},
	}
	// Only set the bridge's MAC address if the kernel version is > 3.3, as it was not supported before that.
	kv, err := kernel.GetKernelVersion()
	if err != nil {
		logrus.Errorf("Failed to check kernel versions: %v. Will not assign a MAC address to the bridge interface", err)
	} else {
		setMac = kv.Kernel > 3 || (kv.Kernel == 3 && kv.Major >= 3)
	}
	/*
	LinkAdd adds a new link device. The type and features of the device are taken fromt the parameters in the link object.
	Equivalent to: `ip link add $link`	
	*/
	i.nlh.LinkAdd(i.Link)
	if setMac {
		//随机生成一个mac地址
		hwAddr := netutils.GenerateRandomMAC()
		if err = i.nlh.LinkSetHardwareAddr(i.Link, hwAddr); err != nil {
			return fmt.Errorf("failed to set bridge mac-address %s : %s", hwAddr, err.Error())
		}
		logrus.Debugf("Setting bridge mac address to %s", hwAddr)
	}
	return err
}
```

#### 2.3.3 initBridgeDriver()

`initBridgeDriver(controller, config)`的实现位于[moby/daemon/daemon_unix.go#L789#L918](https://github.com/moby/moby/blob/17.05.x/daemon/daemon_unix.go#L789#L918)，分析如下:

```go
func initBridgeDriver(controller libnetwork.NetworkController, config *config.Config) error {
	bridgeName := bridge.DefaultBridgeName
	if config.BridgeConfig.Iface != "" {
		bridgeName = config.BridgeConfig.Iface
	}
	netOption := map[string]string{
		bridge.BridgeName:         bridgeName,
		bridge.DefaultBridge:      strconv.FormatBool(true),
		netlabel.DriverMTU:        strconv.Itoa(config.Mtu),
		bridge.EnableIPMasquerade: strconv.FormatBool(config.BridgeConfig.EnableIPMasq),
		bridge.EnableICC:          strconv.FormatBool(config.BridgeConfig.InterContainerCommunication),
	}
	// --ip processing
	if config.BridgeConfig.DefaultIP != nil {
		netOption[bridge.DefaultBindingIP] = config.BridgeConfig.DefaultIP.String()
	}
	var (
		ipamV4Conf *libnetwork.IpamConf
		ipamV6Conf *libnetwork.IpamConf
	)
	ipamV4Conf = &libnetwork.IpamConf{AuxAddresses: make(map[string]string)}
	/*
	It looks for an interface on the OS with the specified name and returns all its IPv4 and IPv6 addresses in CIDR notation.
	If the interface does not exist, it chooses from a predefined list the first IPv4 address which does not conflict with other interfaces on the system.
	*/
	nwList, nw6List, err := netutils.ElectInterfaceAddresses(bridgeName)
	nw := nwList[0]
	if len(nwList) > 1 && config.BridgeConfig.FixedCIDR != "" {
		//将字符串s解析成一个ip地址和子网掩码的结构体中，*net.IPNet
		_, fCIDR, err := net.ParseCIDR(config.BridgeConfig.FixedCIDR)
	}
	// Iterate through in case there are multiple addresses for the bridge
	for _, entry := range nwList {
		if fCIDR.Contains(entry.IP) {
			nw = entry
			break
		}
	}
	//returns the canonical form for the passed network
	ipamV4Conf.PreferredPool = lntypes.GetIPNetCanonical(nw).String()
	//returns the host portion of the ip address identified by the mask
	//返回主机部分网址
	hip, _ := lntypes.GetHostPartIP(nw.IP, nw.Mask)
	//如果ip是全局单播地址，则返回真。
	if hip.IsGlobalUnicast() {
		ipamV4Conf.Gateway = nw.IP.String()
	}
	if config.BridgeConfig.IP != "" {
		ipamV4Conf.PreferredPool = config.BridgeConfig.IP
		ip, _, err := net.ParseCIDR(config.BridgeConfig.IP)
		ipamV4Conf.Gateway = ip.String()
	} else if bridgeName == bridge.DefaultBridgeName && ipamV4Conf.PreferredPool != "" {
		logrus.Infof("Default bridge (%s) is assigned with an IP address %s. Daemon option --bip can be used to set a preferred IP address", bridgeName, ipamV4Conf.PreferredPool)
	}
	if config.BridgeConfig.FixedCIDR != "" {
		_, fCIDR, err := net.ParseCIDR(config.BridgeConfig.FixedCIDR)
		ipamV4Conf.SubPool = fCIDR.String()
	}
	if config.BridgeConfig.DefaultGatewayIPv4 != nil {
		ipamV4Conf.AuxAddresses["DefaultGatewayIPv4"] = config.BridgeConfig.DefaultGatewayIPv4.String()
	}
	//ipv6地址的设置
	var deferIPv6Alloc bool
	if config.BridgeConfig.FixedCIDRv6 != "" {
		_, fCIDRv6, err := net.ParseCIDR(config.BridgeConfig.FixedCIDRv6)
		ones, _ := fCIDRv6.Mask.Size()
		deferIPv6Alloc = ones <= 80
		if ipamV6Conf == nil {
			ipamV6Conf = &libnetwork.IpamConf{AuxAddresses: make(map[string]string)}
		}
		ipamV6Conf.PreferredPool = fCIDRv6.String()
		for _, nw6 := range nw6List {
			if fCIDRv6.Contains(nw6.IP) {
				ipamV6Conf.Gateway = nw6.IP.String()
				break
			}
		}
	}
	if config.BridgeConfig.DefaultGatewayIPv6 != nil {
		if ipamV6Conf == nil {
			ipamV6Conf = &libnetwork.IpamConf{AuxAddresses: make(map[string]string)}
		}
		ipamV6Conf.AuxAddresses["DefaultGatewayIPv6"] = config.BridgeConfig.DefaultGatewayIPv6.String()
	}
	v4Conf := []*libnetwork.IpamConf{ipamV4Conf}
	v6Conf := []*libnetwork.IpamConf{}
	if ipamV6Conf != nil {
		v6Conf = append(v6Conf, ipamV6Conf)
	}
	// Initialize default network on "bridge" with the same name,这里上文已经分析过
	_, err = controller.NewNetwork("bridge", "bridge", "",
		libnetwork.NetworkOptionEnableIPv6(config.BridgeConfig.EnableIPv6),
		libnetwork.NetworkOptionDriverOpts(netOption),
		libnetwork.NetworkOptionIpam("default", "", v4Conf, v6Conf, nil),
		libnetwork.NetworkOptionDeferIPv6Alloc(deferIPv6Alloc))
	return nil
}
```

这个函数的作用大体就是一些基本参数的配置，并在最后新建了一个bridge network


## 3. libnetwork在docker container创建时的工作

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

### 3.3 代码分析---allocateNetwork()

**allocateNetwork()**

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

**allocateNetwork()--->connectToNetwork()**

接下来再分析daemon.connectToNetwork(),该函数位于[moby/daemon/container_operations.go#L684#L796](https://github.com/moby/moby/blob/17.05.x/daemon/container_operations.go#L684#L796)，对于这部分的代码，分析如下：

```go
//以daemon.connectToNetwork(container, netName, epConf.EndpointSettings, updateSettings)为例
func (daemon *Daemon) connectToNetwork(container *container.Container, idOrName string, endpointConfig *networktypes.EndpointSettings, updateSettings bool) (err error) {
	//如果是container模式，返回错误
	if container.HostConfig.NetworkMode.IsContainer() {
		return runconfig.ErrConflictSharedNetwork
	}
	//根据network的name或id查找network，并连接，在3.3.1中再详细分析
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
	//获得容器的sandbox,从controller的Sandbox数组中获取,在3.3.2中分析
	sb := daemon.getNetworkSandbox(container)
	//创建endpoint的配置信息
	createOptions, err := container.BuildCreateEndpointOptions(n, endpointConfig, sb, daemon.configStore.DNS)
	//endPoint的名字
	endpointName := strings.TrimPrefix(container.Name, "/")
	//创建endpoint,下面在3.3.3中分析
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
		//在3.3.2中分析
		sb, err = controller.NewSandbox(container.ID, options...)
		if err != nil {
			return err
		}

		container.UpdateSandboxNetworkSettings(sb)
	}
	//根据network创建endpoint join的配置信息
	joinOptions, err := container.BuildJoinOptions(n)
	//将endpoint加入sandbox中，将在3.3.4分析
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

这部分的工作是
* 1.查找所要connect的network，将network信息更新到container.NetworkSettings.Networks[]中
* 2.获取container的sandbox
* 3.由network创建endpoint
* 4.若2.中获取的sandbox为nil，则由netController创建sandbox
* 5.将endpoint加入sandbox中

#### 3.3.1 network---findAndAttachNetwork()

**connectToNetwork()--->findAndAttachNetwork()**

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

**connectToNetwork()--->findAndAttachNetwork()--->daemon.FindNetwork(idOrName)**

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

**connectToNetwork()--->findAndAttachNetwork()--->daemon.FindNetwork(idOrName)--->daemon.GetNetworkByName(idName)**

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
**connectToNetwork()--->findAndAttachNetwork()--->daemon.FindNetwork(idOrName)--->daemon.GetNetworkByName(idName)--->c.NetworkByName(name)**

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
	//遍历controller中的network
	c.WalkNetworks(s)
	return n, nil
}
```

**connectToNetwork()--->findAndAttachNetwork()--->daemon.FindNetwork(idOrName)--->daemon.GetNetworkByName(idName)--->c.NetworkByName(name)--->c.WalkNetworks(s)**

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

**connectToNetwork()--->findAndAttachNetwork()--->daemon.FindNetwork(idOrName)--->daemon.GetNetworkByName(idName)--->c.NetworkByName(name)--->c.WalkNetworks(s)--->c.Networks()**

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

**connectToNetwork()--->findAndAttachNetwork()--->daemon.FindNetwork(idOrName)--->daemon.GetNetworkByName(idName)--->c.NetworkByName(name)--->c.WalkNetworks(s)--->c.Networks()-->c.getNetworksFromStore()**

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

#### 3.3.2 sandbox---getNetworkSandbox()、NewSandbox()

下面分别在3.3.2.1和3.3.2.2分析`getNetworkSandbox()`和`NewSandbox()`函数。

**3.3.2.1 connectToNetwork()--->getNetworkSandbox()**

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

**connectToNetwork()--->getNetworkSandbox()--->daemon.netController.WalkSandboxes()**

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

**connectToNetwork()--->getNetworkSandbox()--->daemon.netController.WalkSandboxes()--->c.Sandboxes()**

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

**3.3.2.2 connectToNetwork()--->NewSandbox()**

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
			/*
				NewSandbox provides a new sandbox instance created in an os specific way  provided a key which uniquely identifies the sandbox
				下面继续分析
			*/
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

**connectToNetwork()--->NewSandbox()--->osl.NewSandbox()**

接着分析`osl.NewSandbox(sb.Key(), false, false)`,它的实现位于[moby/vendor/github.com/docker/libnetwork/osl/namespace_linux.go#L196#L238](https://github.com/moby/moby/blob/17.05.x/vendor/github.com/docker/libnetwork/osl/namespace_linux.go#L196#L238)，主要代码为：

```go
func NewSandbox(key string, osCreate, isRestore bool) (Sandbox, error) {
	//根据上文可以知道，这里isRestore是false，所以会执行下面的代码
	if !isRestore {
		//创建network namespace，下面接着分析
		err := createNetworkNamespace(key, osCreate)
		if err != nil {
			return nil, err
		}
	} else {
		//createBasePath()函数只执行一次
		once.Do(createBasePath)
	}
	n := &networkNamespace{path: key, isDefault: !osCreate, nextIfIndex: make(map[string]int)}
	//get network namespace
	sboxNs, err := netns.GetFromPath(n.path)
	//create a netlink handle
	n.nlHandle, err = netlink.NewHandleAt(sboxNs, syscall.NETLINK_ROUTE)
	//set the timeout on the sandbox netlink handle sockets
	n.nlHandle.SetSocketTimeout(ns.NetlinkSocketsTimeout)
	
	// As starting point, disable IPv6 on all interfaces
	if !n.isDefault {
		//disable IPv6 on all interfaces on network namespace
		setIPv6(n.path, "all", false)
	}
	//set up lookback
	n.loopbackUp()
	//返回该namespace
	return n, nil
}
```

**connectToNetwork()--->NewSandbox()--->osl.NewSandbox()-->createNetworkNamespace()**

接着分析`createNetworkNamespace(key, osCreate)`，它的实现位于[moby/vendor/github.com/docker/libnetwork/osl/namespace_linux.go#L302#L322](https://github.com/moby/moby/blob/17.05.x/vendor/github.com/docker/libnetwork/osl/namespace_linux.go#L302#L322)，主要代码为：

```go
func createNetworkNamespace(path string, osCreate bool) error {
	//创建namespace文件
	createNamespaceFile(path)
	cmd := &exec.Cmd{
		//Returns "/proc/self/exe"，代表当前程序
		Path:   reexec.Self(),
		Args:   append([]string{"netns-create"}, path),
		Stdout: os.Stdout,
		Stderr: os.Stderr,
	}
	//从上文可以看到，osCreate为false，所以下面这段代码不执行
	if osCreate {
		cmd.SysProcAttr = &syscall.SysProcAttr{}
		cmd.SysProcAttr.Cloneflags = syscall.CLONE_NEWNET
	}
	//namespace creation reexec command
	cmd.Run()
	return nil
}	
```

#### 3.3.3 endpoint---n.CreateEndpoint()

**connectToNetwork()--->n.CreateEndpoint()**

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
	/*
	Initialize ep.network with a possibly stale copy of n. We need this to get network from store. But once we get it from store we will have the most uptodate copy possibly.
	ep.network初始化赋值为n，是为了之后从store从获取network，从store中获取的network是最新的版本
	*/	
	ep.network = n
	ep.locator = n.getController().clusterHostID()
	//根据n，从store中获取最新的network
	ep.network, err = ep.getNetworkFromStore()     
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
	//给endpoint分配ipv4和ipv6地址
	if err = ep.assignAddress(ipam, true, n.enableIPv6 && !n.postIPv6); err != nil {
		return nil, err
	}
	//向network中加入endpoint，在下文中继续分析
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

下面将分析`n.addEndpoint(ep)`函数。

**connectToNetwork()--->n.CreateEndpoint()--->n.addEndpoint(ep)**

`n.addEndpoint(ep)`的实现位于[moby/vendor/github.com/docker/libnetwork/network.go#L874#L887](https://github.com/moby/moby/blob/17.05.x/vendor/github.com/docker/libnetwork/network.go#L874#L887)，主要代码为：

```go
func (n *network) addEndpoint(ep *endpoint) error {
	//获得网络驱动,下面3.3.3.1继续分析
	d, err := n.driver(true)
	//调用网络驱动创建endpoint，有bridge、host等,下面3.3.3.2继续分析
	err = d.CreateEndpoint(n.id, ep.id, ep.Interface(), ep.generic)
	return nil
}
```

下面接着看3.3.3.1的`n.driver(true)`和3.3.3.2的`d.CreateEndpoint()`

**3.3.3.1 connectToNetwork()--->n.CreateEndpoint()--->n.addEndpoint(ep)--->n.driver(true)**

`n.driver(true)`的实现位于[moby/vendor/github.com/docker/libnetwork/network.go#L760#L781](https://github.com/moby/moby/blob/17.05.x/vendor/github.com/docker/libnetwork/network.go#L760#L781)，主要代码为：

```go
func (n *network) driver(load bool) (driverapi.Driver, error) {
	//根据网络类型从DrvRegistry中获取网络驱动，如果没有，则看load是否为true，是true的话，就载入该网络类型的驱动
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

**3.3.3.2 connectToNetwork()--->n.CreateEndpoint()--->n.addEndpoint(ep)--->d.CreateEndpoint()**

现在看`d.CreateEndpoint(n.id, ep.id, ep.Interface(), ep.generic)`调用，d即driver的种类有：bridge、host、ipvlan、macvlan、overlay、remote六种，每种驱动都有`CreateEndpoint()`的实现，这里我们先来看看brdige驱动的`CreateEndpoint()`，实现位于[moby/vendor/github.com/docker/libnetwork/drivers/bridge/bridge.go#L902#L1081](https://github.com/moby/moby/blob/17.05.x/vendor/github.com/docker/libnetwork/drivers/bridge/bridge.go#L902#L1081)，主要代码为：

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
	/*
	添加veth pair
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
		//设置sandbox端veth的MTU
		err = d.nlh.LinkSetMTU(sbox, config.Mtu)
	}
	// Attach host side pipe interface into the bridge，将host veth加入docker0,之后分析
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
	//Store the sandbox side pipe interface parameters，这之后会调用ep.Join()将container的sandbox和endpoint结合
	endpoint.srcName = containerIfName
	endpoint.macAddress = ifInfo.MacAddress()
	endpoint.addr = ifInfo.Address()
	endpoint.addrv6 = ifInfo.AddressIPv6()
	// Set the sbox's MAC if not provided. If specified, use the one configured by user, otherwise generate one based on IP.
	if endpoint.macAddress == nil {
		endpoint.macAddress = electMacAddress(epConfig, endpoint.addr.IP)
		ifInfo.SetMacAddress(endpoint.macAddress)
	}
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

下面继续分析`addToBridge()`。

**connectToNetwork()--->n.CreateEndpoint()--->n.addEndpoint(ep)--->d.CreateEndpoint()--->addToBridge()**

这里再看看`addToBridge(d.nlh, hostIfName, config.BridgeName)`，实现位于[moby/vendor/github.com/docker/libnetwork/drivers/bridge/bridge.go#L848#L869](https://github.com/moby/moby/blob/17.05.x/vendor/github.com/docker/libnetwork/drivers/bridge/bridge.go#L848#L869)，主要代码为：

```go
func addToBridge(nlh *netlink.Handle, ifaceName, bridgeName string) error {
	//finds a link by name and returns a pointer to the object.
	//获得host端veth
	link, err := nlh.LinkByName(ifaceName)
	/*
	LinkSetMaster sets the master of the link device.
	add host link to bridge via netlink.
	Equivalent to: `ip link set $link master $master`
	下面详细再看看
	*/
	nlh.LinkSetMaster(link,&netlink.Bridge{LinkAttrs: netlink.LinkAttrs{Name: bridgeName}})
	return nil
}
```
**connectToNetwork()--->n.CreateEndpoint()--->n.addEndpoint(ep)--->d.CreateEndpoint()--->addToBridge()--->nlh.LinkSetMaster()**

接着到`nlh.LinkSetMaster(link,&netlink.Bridge{LinkAttrs: netlink.LinkAttrs{Name: bridgeName}})`的调用，它的实现位于[moby/vendor/github.com/vishvananda/netlink/link_linux.go#L380#L391](https://github.com/moby/moby/blob/17.05.x/vendor/github.com/vishvananda/netlink/link_linux.go#L380#L391)，主要代码为：

```go
func (h *Handle) LinkSetMaster(link Link, master *Bridge) error {
	index := 0
	if master != nil {
		//返回一个LinkAttrs结构体，LinkAttrs represents data shared by most link types，这个结构体包含了MTU、TxQLen等一些属性
		masterBase := master.Attrs()
		//获取正确的link index
		h.ensureIndex(masterBase)
		//获得bridge的下标
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

**connectToNetwork()--->n.CreateEndpoint()--->n.addEndpoint(ep)--->d.CreateEndpoint()--->addToBridge()--->nlh.LinkSetMaster()--->h.LinkSetMasterByIndex(link, index)**

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

目前将host veth加入到对应的bridge中（即docker0），就先看到这里，再接下去的地方好像太底层了。

#### 3.3.4 ep.Join()

**connectToNetwork()--->ep.Join()**

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
**connectToNetwork()--->ep.Join()--->ep.sbJoin()**

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
	//调用网络驱动的Join函数，endpoint加入到sandbox中，将在下文中分析
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

**connectToNetwork()--->ep.Join()--->ep.sbJoin()--->d.Join()**

`d.Join(nid, epid, sb.Key(), ep, sb.Labels())`分析bridge驱动下的实现，位于[moby/vendor/github.com/docker/libnetwork/drivers//bridge/bridge.go#L1204#L1247](https://github.com/moby/moby/blob/17.05.x/vendor/github.com/docker/libnetwork/drivers//bridge/bridge.go#L1204#L1247)，主要代码为：

```go
func (d *driver) Join(nid, eid string, sboxKey string, jinfo driverapi.JoinInfo, options map[string]interface{}) error {
	// InitOSContext initializes OS context while configuring network resources
	defer osl.InitOSContext()()
	network, err := d.getNetwork(nid)
	endpoint, err := network.getEndpoint(eid)
	//从options中获得containerConfig，这里的options是sb.Labels()，返回的是sb.config.generic
	endpoint.containerConfig, err = parseContainerOptions(options)
	// InterfaceName returns an InterfaceNameInfo go interface to facilitate setting the names for the interface.
	// 返回一个InterfaceNameInfo的接口，用于名字的设置,返回的是ep.iface,类型endpointInterface
	iNames := jinfo.InterfaceName()
	// defaultContainerVethPrefix = "eth" 
	containerVethPrefix := defaultContainerVethPrefix
	if network.config.ContainerIfacePrefix != "" {
		containerVethPrefix = network.config.ContainerIfacePrefix
	}
	//设置endpoint srcName为containerVethPrefix
	iNames.SetNames(endpoint.srcName, containerVethPrefix)
	// SetGateway sets the default IPv4 gateway when a container joins the endpoint.
	err = jinfo.SetGateway(network.bridge.gatewayIPv4)
	// SetGatewayIPv6 sets the default IPv6 gateway when a container joins the endpoint.
	err = jinfo.SetGatewayIPv6(network.bridge.gatewayIPv6)
	return nil
}
```

目前该函数中存在一些疑点，该段函数没有看到sboxKey的使用，endpoint是如何加入sandbox的，还不是很清楚。

## 结语

本文分析了libnetwork的工作流程，分别从docker daemon初始化时和docker container创建时两方面来分析。docker daemon初始化时，对netController、Network进行了创建；docker container创建时，创建了container的sandbox、endpoint并将endpoint加入sandbox中。本文分别对上述过程进行了简要的分析。

## 参考

* *[Docker 网络部分执行流分析（libnetwork源码解读）](http://blog.csdn.net/gao514916467/article/details/51242299)*
* *[从docker daemon的角度，添加了userland-proxy的起停开关](http://www.dataguru.cn/thread-544489-1-1.html)*
* *[Linux Bridge的IP NAT细节探析-填补又一坑的过程](http://www.bubuko.com/infodetail-1994270.html)*
* *[浅谈Go语言中的结构体struct & 接口Interface & 反射](http://www.jb51.net/article/119585.htm)*
* *[Docker Embedded DNS](http://www.jianshu.com/p/4433f4c70cf0)*
