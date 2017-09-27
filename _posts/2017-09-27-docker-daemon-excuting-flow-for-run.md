---
layout:     post
title:      " docker源码阅读之二---docker daemon启动流程 "
date:       2017-9-27 18:00:00 
author:     "guanjunjian"
categories: Docker源码阅读
tags:
    - docker_run
    - network
---

* content
{:toc}

> 上文分析了docker client段对于docker run命令的处理，client将create和start命令发送给daemon;
> 
> 本文主要分析daemon的启动过程，以及对create和start命令的处理;
>  
> 源码阅读基于docker [version1.17.05.x](https://github.com/moby/moby/tree/17.05.x)。




## 1. docker daemon的入口main

### 1.1 源码

docker daemon的main函数位于[moby/cmd/dockerd/docker.go](https://github.com/moby/moby/blob/17.05.x/cmd/dockerd/docker.go#L100#L122)，代码的主要内容是：

```go
func main() {
	//看有没有注册的初始化函数，如果有，就直接return,这个函数似乎与dockerinit有关
	if reexec.Init() {
		return
	}
	//构建一个docker服务器命令行接口对象，命令行接口包含了docker服务器所有可以执行的命令，并通过每一个命令结构体对象中的Run等成员函数来具体执行
	cmd := newDaemonCommand() 
	cmd.Execute()
```

`newDaemonCommand()`调用的地方：1.cmd/dockerd/docker.go的main函数；2.cmd/docker/docker.go的`newDockerCommand()`中也有调用，这里的调用是为了启动daemon。

### 1.2 流程图

![](/img/in-post/post-docker-daemon-excuting-flow-for-run/docker-daemon-main.png)



### 2. newDaemonCommand()

#### 2.1  newDaemonCommand()主代码

newDaemonCommand()包含daemon初始化的流程，主要代码为:

```go
func newDaemonCommand() *cobra.Command {
	//获取配置信息
	opts := daemonOptions{
		daemonConfig: config.New(),
		common:       cliflags.NewCommonOptions(),
	}

	//docker daemon命令行对象，与docker client中的相似
	cmd := &cobra.Command{
		Use:           "dockerd [OPTIONS]",
		Short:         "A self-sufficient runtime for containers.",
		SilenceUsage:  true,
		SilenceErrors: true,
		Args:          cli.NoArgs,
		RunE: func(cmd *cobra.Command, args []string) error {
			opts.flags = cmd.Flags()
			//daemon command执行时，执行该函数
			return runDaemon(opts)
		},
	}
	cli.SetupRootCommand(cmd)

	//解析命令
	flags := cmd.Flags()
	//设置docker daemon启动的时候是否使用了version等一些命令
	flags.BoolVarP(&opts.version, "version", "v", false, "Print version information and quit")
	flags.StringVar(&opts.configFile, "config-file", defaultDaemonConfigFile, "Daemon configuration file")
	opts.common.InstallFlags(flags)
	installConfigFlags(opts.daemonConfig, flags)
	installServiceFlags(flags)

	return cmd
}
```

### 2.2 runDaemon(opts)

runDaemon是在daemon command使用时被调用，主要代码为

```go
func runDaemon(opts daemonOptions) error {
	daemonCli := NewDaemonCli() //创建daemon客户端对象
	daemonCli.start(opts) ////启动daemonCli
}
```

### 2.2.1 NewDaemonCli()

`NewDaemonCli()`创建一个DaemonCli结构体对象，该结构体包含配置信息，配置文件，参数信息，APIServer,Daemon对象，authzMiddleware（认证插件），代码如下:

```go
type DaemonCli struct {
	*config.Config  //配置信息
	configFile *string //配置文件
	flags      *pflag.FlagSet  //flag参数信息
	api             *apiserver.Server //APIServer:提供api服务，定义在docker/api/server/server.go
	d               *daemon.Daemon  //Daemon对象,结构体定义在daemon/daemon.go文件中
	authzMiddleware *authorization.Middleware // authzMiddleware enables to dynamically reload the authorization plugins
}

```

其中APIServer在接下来的daemonCli.start()实现过程中具有非常重要的作用。apiserver.Server的结构体为：

```go
// Server contains instance details for the server
type Server struct {
	cfg           *Config//apiserver的配置信息
	servers       []*HTTPServer//httpServer结构体对象，包括http.Server和net.Listener监听器。
	routers       []router.Router//路由表对象Route,包括Handler,Method, Path
	routerSwapper *routerSwapper//路由交换器对象，使用新的路由交换旧的路由器
	middlewares   []middleware.Middleware//中间件
}
```

### 2.2.2 daemonCli.start(opts) 

daemonCli.start(opts)的主要代码以实现如下：

```go
func (cli *DaemonCli) start(opts daemonOptions) (err error) {
	//1.设置默认可选项参数
	opts.common.SetDefaultOptions(opts.flags)

	//2.根据opts对象信息来加载DaemonCli的配置信息config对象，并将该config对象配置到DaemonCli结构体对象中去  
	cli.Config, err = loadDaemonCliConfig(opts)

	//3.对DaemonCli结构体中的其它成员根据opts进行配置
	cli.configFile = &opts.configFile
	cli.flags = opts.flags

	//4.根据DaemonCli结构体对象中的信息定义APIServer配置信息结构体对象&apiserver.Config(包括tls传输层协议信息)
	serverConfig := &apiserver.Config{
		Logging:     true,
		SocketGroup: cli.Config.SocketGroup,
		Version:     dockerversion.Version,
		EnableCors:  cli.Config.EnableCors,
		CorsHeaders: cli.Config.CorsHeaders,
	}

	//5.根据定义好的&apiserver.Config新建APIServer对象，并赋值到DaemonCli实例的对应属性中
	api := apiserver.New(serverConfig)
	cli.api = api

	for i := 0; i < len(cli.Config.Hosts); i++ {
		//6.解析host文件及传输协议（tcp）等内容
		cli.Config.Hosts[i], err = dopts.ParseHost(cli.Config.TLS, cli.Config.Hosts[i])	
		protoAddr := cli.Config.Hosts[i]
		protoAddrParts := strings.SplitN(protoAddr, "://", 2)
		proto := protoAddrParts[0]
		addr := protoAddrParts[1]
		
		//7.根据host解析内容初始化监听器listener.Init()
		ls, err := listeners.Init(proto, addr, serverConfig.SocketGroup, serverConfig.TLSConfig)
		ls = wrapListeners(proto, ls)

		//8.为建立好的APIServer设置我们初始化的监听器listener，可以监听该地址的连接
		api.Accept(addr, ls...)
	}

	//9.根据DaemonCli.Config.ServiceOptions来注册一个新的服务对象
	registryService := registry.NewService(cli.Config.ServiceOptions)

	//10.根据DaemonCli中的相关信息来新建libcontainerd对象 
	containerdRemote, err := libcontainerd.New(cli.getLibcontainerdRoot(), cli.getPlatformRemoteOptions()...)
	
	/*
	11.设置信号捕获,进入Trap()函数，可以看到os.Interrupt, syscall.SIGTERM, syscall.SIGQUIT, syscall.SIGPIPE
	四种信号，这里传入一个cleanup()函数，当捕获到这四种信号时，可以利用该函数进行shutdown善后处理
	*/
	signal.Trap(func() {
		cli.stop()
		<-stopc // wait for daemonCli.start() to return处于阻塞状态，等待stopc通道返回数据
	})

	//12.提前通知系统api可以工作了，但是要在daemon安装成功之后
	preNotifySystem()

	//13.根据DaemonCli的配置信息，注册的服务对象及libcontainerd对象来构建Daemon对象
	d, err := daemon.NewDaemon(cli.Config, registryService, containerdRemote, pluginStore)

	//14.新建cluster对象
	c, err := cluster.New(cluster.Config{
		Root:                   cli.Config.Root,
		Name:                   name,
		Backend:                d,
		NetworkSubnetsProvider: d,
		DefaultAdvertiseAddr:   cli.Config.SwarmDefaultAdvertiseAddr,
		RuntimeRoot:            cli.getSwarmRunRoot(),
	})
	d.SetCluster(c)

	//15.重启Swarm容器
	d.RestartSwarmContainers()

	//16.将新建的Daemon对象与DaemonCli相关联
	cli.d = d

	//17.初始化路由器
	initRouter(api, d, c)

	//18.新建goroutine来监听apiserver执行情况，当执行报错时通道serverAPIWait就会传出错误信息
	go api.Wait(serveAPIWait)

	//19.通知系统Daemon已经安装完成，可以提供api服务了
	notifySystem()

	//20.等待apiserver执行出现错误，没有错误则会阻塞到该语句，直到server API完成
	errAPI := <-serveAPIWait

	//21.执行到这一步说明，serverAPIWait有错误信息传出（一下均是），所以对cluster进行清理操作
	c.Cleanup()

	//22.关闭Daemon
	shutdownDaemon(d)

	//23.闭libcontainerd
	containerdRemote.Cleanup()
}
```

这部分代码实现了daemonCli、apiserver的初始化，其中apiserver的功能是处理client段发送请求，并将请求路由出去。所以`initRouter(api, d, c)`中包含了api路由的具体实现，将会在下文中分析。

## 结语

本文介绍了docker daemon到serverapi的初始化过程，下文将分析serverapi如何将docker run命令路由到具体的执行函数。

## 参考

* *[docker源码阅读笔记三---Xubo](http://blog.xbblfz.site/2017/04/20/docker%E5%AE%A2%E6%88%B7%E7%AB%AF%E4%B8%8E%E6%9C%8D%E5%8A%A1%E5%99%A8%E7%AB%AF%E9%80%9A%E4%BF%A1%E6%A8%A1%E5%9D%97%E4%B8%80/)*

