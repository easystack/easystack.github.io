---
title: "记一次kubelet 内存泄漏"
date: 2021-11-09T15:50:43+08:00
draft: false
description: "kubelet memroy leak"
author: "yylt"
categories: ["container","problem"]
tags: ["containerd","kubelete"]
---


# 背景

一次生产环境中，发现 kubelet 内存达到上 GB 以上，这个不符合平常使用情况，首先想到的是内存泄漏，那么肯定使用 pprof 以及调查为什么会产生内存泄漏

# profile

## 基础
- cpu `/debug/pprof/profile`，主要分析耗时和优化算法，得到 profile 文件
- heap: `/debug/pprof/heap`，查看活动对象的内存分配情况，得到 profile 文件
- threadcreate: `/debug/pprof/threadcreate`, 线程创建概况报告程序中导致创建新的操作系统线程的部分
- goroutine： 报告所有当前 goroutine 的堆栈信息，**没有 profile 文件**
- trace: `/debug/pprof/trace` 当前程序的执行跟踪，go tool trace 中使用

# kubelet
查看平台的内存使用情况，如下
注：（以下数据非当时环境，是之后重新复现后取的，比发生泄漏环境要低的多)

```shell
# top
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
31599 root      20   0 2877396 983.5m  66424 S   4.3  6.2  28:00.93 kubelet           
```

kubelet 数据默认打开 pprof， 通过 [pprof](https://github.com/google/pprof) 拿到数据 , 通过以下命令方式

```shell
# pprof -tls_ca {cafile} -tls_key {keyfile} -tls_cert {certfile}
 https://{host}:10250/debug/pprof/heap
```

拿到数据后，还可以拿到正常节点的数据做对比，如下

正常 kubelet 内存使用
![](https://s2.loli.net/2022/04/21/SLKZQjG9Yvdx4Wk.png)

不正常 kubelet 内存使用
![](https://s2.loli.net/2022/04/21/jCefqQ7JARYrSno.png)

## 基础
描述下 kubelet 当前目录和会涉及的代码片段
```
pkg/kubelet/
├── cri #cri接口
├── server # http hanlder入口
├── stats # 容器 cpu/memory，filesystem info 抓取
├── kuberuntime # 桥接容器运行时 和 kubelet操作
└── cm #container manager缩写，控制器内容包括 cgroup，topology,device等
```

可以直接跨越到 cri/ 内查看 当前接口实现，未设置超时时间
```go
func (r *remoteRuntimeService) ListContainerStats(filter *runtimeapi.ContainerStatsFilter) ([]*runtimeapi.ContainerStats, error) {
	// Do not set timeout, because writable layer stats collection takes time.
	// TODO(random-liu): Should we assume runtime should cache the result, and set timeout here?
	ctx, cancel := getContextWithCancel()
	defer cancel()
	// 这里未设置超时时间
	resp, err := r.runtimeClient.ListContainerStats(ctx, &runtimeapi.ListContainerStatsRequest{
		Filter: filter,
	})
```

当前调用链大致如下 , 可以看到及时客户端退出，但是 kubelet 到 containerd 的连接还是会保持，直到拿到数据为止
![](https://s2.loli.net/2022/04/26/bM1swGoBIDTx3gF.png)

那么考虑就是如何复现这个问题，但通过测试代码 [1] 并未复现问题，这里其实一直是阻塞调查的地方，那么不妨换个思路，现继续挖下为什么 containerd 没有返回数据呢？

到这里，其实可以发现 kubelet 都集中在 `stats.ListPodStats` 花费上，该调用会最终到 containerd 的 Stats 接口上，那么应该继续分析 containerd ，这应该是产生问题的根本原因

# containerd

通过抓取 heap 和 goroutine 信息分析，而我们环境中 containerd 的 debug socket 已经打开，所以比较方便拿到 containerd profile 信息

```shell
# ctr pprof heap > heap.profile
# ctr pprof goroutines > goroutine.profile
```

拿到数据后，还可以拿到正常节点的数据做对比

正常 containerd 内存使用
![](https://s2.loli.net/2022/04/21/e1gxopz2HduOiQM.png)

不正常 containerd 内存使用
![](https://s2.loli.net/2022/04/21/qSt6okRhXIEYWUZ.png)

看到的是 SPDY 内存消耗较多，我们知道 SPDY 是 HTTP/2 前身，主要用于流式连接，当前主要是 接口 exec, attach, portfoward 在接口上，以下是简单对 exec 的代码理解 , 然后使用了 exec 确能稳定复现问题，复现代码如下

```sh
#!/bin/bash
i=0
NUM=$1
while [ "$i" -lt $NUM ]; do
   (kubectl exec -i test-74cf75b654-gw5hk -- ls /opt)&
   let "i += 1"
done
```

## exec 流程

流程需要从 kubelet 开始分析 , 通过前文中对 kubelet 目录的简单介绍，那么入口肯定是在 server 目录内，直接到主题 Exec，对应函数是   `getExec(request *restful.Request, response *restful.Response)` ， 行为简单描述如下
	- 校验参数，通常 stdout/stderr 为 true，目前常用的是 `-it`，也就是需要配置 stdin 和 ttry 是否为 true
	- 查询 pod 当前是否支持 exec
	- 执行 GetExec 获取 url
	- 创建代理，转发 apiserver 数据 stdin 和 stdout 

cri 服务 主要分为两部分，重点描述 ServerExec
	- GetExec：创建 url 路由，增加 token  `host:port/exec/{token}`
	- ServerExec： 创建 task 和 process，并绑定标准输入和输出等

ServerExec
- 升级 http 为 SPDY stream，看上去 spdy 是比较重，因为要至少建立三个 goroutine `handler.waitForStreams(streamCh, expectedStreams, expired.C)`
- 执行 Exec (在 streamRuntime 内)，等待 process 结束，或者上游退出

```go
//创建Task， task实际是 containerd/containerd/task.go 类型
task, err := container.Task(ctx, nil)
if err != nil {
	return nil, errors.Wrap(err, "failed to load task")
}
pspec := spec.Process
pspec.Args = opts.cmd
pspec.Terminal = opts.tty
if opts.tty {
	oci.WithEnv([]string{"TERM=xterm"})(ctx, nil, nil, spec)
}

volatileRootDir := c.getVolatileContainerRootDir(id)
var execIO *cio.ExecIO
// 创建process, 实际是 containerd/containerd/process.go 类型
process, err := task.Exec(ctx, execID, pspec,
	func(id string) (containerdio.IO, error) {
		var err error
		execIO, err = cio.NewExecIO(id, volatileRootDir, opts.tty, opts.stdin != nil)
		return execIO, err
	},
)
// 获取 process 退出管道
exitCh, err := process.Wait(ctx)
err := process.Start(ctx)

// 将execIO 和 http 流绑定
// 内部会创建协程 stdin ，stdout, waitGroup
attachDone := execIO.Attach(cio.AttachOptions{
	Stdin:     opts.stdin,
	Stdout:    opts.stdout,
	Stderr:    opts.stderr,
	Tty:       opts.tty,
	StdinOnce: true,
	CloseStdin: func() error {
		return process.CloseIO(ctx, containerd.WithStdinCloser)
	},
})

select {
case <-execCtx.Done():
case exitRes := <-exitCh:
}
```

这里是阻塞在调用 exec 上，但是和 cpuAndMemoryStats 并无关系，如果是有关系，那么也只有在 shim 这一级，因为都要调用 shim 的 rpc 接口

# shim
当前 runc 使用的 shim 是  containerd-shim-runc-v2, 在内置的 shim server 上有一个方便调试的方式，发送 USER1 信号 可以获取 goroutines 信息，具体代码如下

```go
// runtime/v2/shim/shim_unix.go
func setupDumpStacks(dump chan<- os.Signal) {
	signal.Notify(dump, syscall.SIGUSR1)
}
```

```sh
// 发送 USER1 信号
kill -10 {pid}
// 获取containerd 日志，获取BEGIN goroutine stack dump 日志
journalctl -eu containerd >contaienrd.log
```

查看 shim 的 goroutines 信息，因为和 ttrpc 有关，那直接找 ttrpc 信息吧，可以发现以下信息，这里显然不应该有那么多 goroutine hang 在 `vendor/github.com/containerd/ttrpc/server.go:444 ` 行，那么继续看下 ttrpc 有关实现呢
```
$ cat shim.goroutine.profile |grep ttrpc/server | sort |uniq -c |sort
      1 	vendor/github.com/containerd/ttrpc/server.go:362 +0x149
      1 	vendor/github.com/containerd/ttrpc/server.go:404 +0x5ee
      1 	vendor/github.com/containerd/ttrpc/server.go:431 +0x41a
      1 	vendor/github.com/containerd/ttrpc/server.go:459 +0x6bd
      1 	vendor/github.com/containerd/ttrpc/server.go:87 +0x107
      2 	vendor/github.com/containerd/ttrpc/server.go:127 +0x2a7
      2 	vendor/github.com/containerd/ttrpc/server.go:332 +0x2ce
      6 	vendor/github.com/containerd/ttrpc/server.go:438 +0xf2
    164 	vendor/github.com/containerd/ttrpc/server.go:444 +0x245
    169 	vendor/github.com/containerd/ttrpc/server.go:434 +0x63f
```

## ttrpc
**reqCh, respCh, msgCh 均为非缓存 channel**

服务端大致如下
![](https://s2.loli.net/2022/04/21/AwBVbSm9kuL2U1h.png)

- 每创建一个连接， 都会产生 2 个协程来处理会话
- recv 协程： 从客户端读取数据，并校验合法性写入 Channel 中，交给 worker 来处理
- worker 协程：收到请求后，**并发创建协程**调用注册的服务

1.0.1 版本 客户端大致如下
![](https://s2.loli.net/2022/04/21/WjuCD8eSObodZwY.png)
- 和服务端类似，工作协程同时处理接收和发送请求，
- 重点是 **发送和接收是在一个协程中处理，并通过内部 waitCall 来同步数据**

发生死锁的情况是 客户端 阻塞在 send 过程，此时因为无法处理返回的 Resp 信息，继而导致服务端的应答数据阻塞在 net write buffer 中
什么情况会导致 send 阻塞，网络发送过程有以下情况 , 进程将数据拷贝到内核缓存区，之后由软中断发送出去，该控制写缓存大小为 tcp_wmem, 内核参数配置为 `4096  16384   4194304`

针对上述导致死锁的情况，有一个相关 patch[2] 解决，该 patch 修改方式如下图

1.1.0 版本 客户端
![](https://s2.loli.net/2022/04/21/Uc5GHopBs1brWDA.png)

发送和接收 都通过不同的协程处理，不再出现竞争情况发生

# 参考

[1]. https://gist.github.com/yylt/0d3f2d554fa7eddd9cafe406ef0c9d75
[2]. https://github.com/containerd/ttrpc/pull/94
