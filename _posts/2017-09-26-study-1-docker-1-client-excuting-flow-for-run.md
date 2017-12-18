---
layout:     post
title:      "study1.docker源码阅读之一---docker client命令行执行流程 "
date:       2017-9-26 22:40:00 
author:     "guanjunjian"
categories: Docker源码阅读
tags:
    - docker_run
    - network
    - study
---

* content
{:toc}

> 开始阅读docker源码，最终目的是了解docker network的实现;
> 
> 本文从*docker run --net bridge ubuntu* 追踪docker network初始化的过程，同时了解docker对网络数据包的处理流程;
>  
> 源码阅读基于docker [version1.17.05.x](https://github.com/moby/moby/tree/17.05.x)。




## 1. docker client的入口main

### 1.1 源码

docker client的main函数位于[moby/cmd/docker/docker.go](https://github.com/moby/moby/blob/17.05.x/cmd/docker/docker.go#L161#184)，代码的主要内容是：

```go
func main() {
	...
	dockerCli := command.NewDockerCli(stdin, stdout, stderr)
	cmd := newDockerCommand(dockerCli)
	if err := cmd.Execute(); 
	...
}
```

**这部分代码的主要工作是：**

*  生成一个带有输入输出的客户端对象

*  根据dockerCli客户端对象，解析命令行参数，生成带有命令行参数及客户端配置信息的cmd命令行对象

*  根据输入参数args完成命令执行

### 1.2 流程图

![](/img/study/study-1-docker-1-client-excuting-flow-for-run/docker-client-main.png)

我们需要追踪的是docker run的执行流程，从流程图中可以看到，从`cmd := newDockerCommand(dockerCli)`的一步步进行下去，可以追踪到`runContainer(dockerCli,opts,copts,contianerConfig)`，而这里就是run命令的执行函数。
在具体了解`runContainer()`的具体执行之前，先了解一下`cobra.Command`结构体。

### 1.3 cobra.Command

#### 1.3.1 首先介绍cobra这个库的简单使用:

```go
package main

import (
    "fmt"

    "github.com/spf13/cobra"
)

func main() {
   //1.定义主命令
    var Version bool
    var rootCmd = &cobra.Command{
        Use:   "root [sub]",
        Short: "My root command",
       //命令执行的函数
        Run: func(cmd *cobra.Command, args []string) {
            fmt.Printf("Inside rootCmd Run with args: %v\n", args)
            if Version {
             fmt.Printf("Version:1.0\n")
            }
        },


    }
   //2.定义子命令
    var subCmd = &cobra.Command{
        Use:   "sub [no options!]",
        Short: "My subcommand",
        //命令执行的函数
        Run: func(cmd *cobra.Command, args []string) {
            fmt.Printf("Inside subCmd Run with args: %v\n", args)
        },

    }
    //添加子命令
    rootCmd.AddCommand(subCmd)
   //3.为命令添加选项
    flags := rootCmd.Flags()
    flags.BoolVarP(&Version, "version", "v", false, "Print version information and quit")
      //执行命令
    _ = rootCmd.Execute()
}

```

基本用法大概就是四步：
* 定义一个主命令（包含命令执行函数等） 
* 定义若干子命令（包含命令执行函数等，根据需要可以为子命令定义子命令），并添加到主命令
* 为命令添加选项
* 执行命令

#### 1.3.2 docker中cobra.Command的使用

docker中`Command`的使用就体现在`commands.AddCommands()`。在`commands.AddCommands(cmd, dockerCli)`中，将run、build等一些命令的执行函数添加到commands中。

## 2. runContainer()

### 2.1 源码

runContainer()的代码位于[moby/cli/command/container/run.go](https://github.com/moby/moby/blob/17.05.x/cli/command/container/run.go#L96#L269),代码的主要部分为：

```go
func runContainer(dockerCli *command.DockerCli, opts *runOptions, copts *containerOptions, containerConfig *containerConfig) error {
	config := containerConfig.Config
	hostConfig := containerConfig.HostConfig
	...
	createResponse, err := createContainer(ctx, dockerCli, containerConfig, opts.name)  //向daemon发送create
	...
	client.ContainerStart(ctx, createResponse.ID, types.ContainerStartOptions{}) ////向daemon发送post

}
```

### 2.2 流程图

![](/img/study/study-1-docker-1-client-excuting-flow-for-run/docker-client-runContainer.png)

在`ContainerCreate()`和`ContainerStart()`中分别向daemon发送了create和start命令。下一步，就需要到docker daemon中分析daemon对create和start的处理。

## 结语

本文分析了docker client对于docker run命令的处理流程，从上文的分析中的最后可以看到，client向daemon分别发送了create和start两个命令，这是容易创建的两个步骤，在下一篇文章中将分析daemon的处理流程。

## 参考

* *[docker源码阅读笔记二---Xubo](http://blog.xbblfz.site/2017/04/18/docker%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB%E7%AC%94%E8%AE%B0%E4%BA%8C/)*
* *[docker命令解析](http://blog.csdn.net/idwtwt/article/details/52733235)*
