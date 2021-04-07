---
title: "使用 Go 每分钟处理百万请求 "
date: 2021-04-05T22:25:52+08:00 
toc: true
tags:
 - go

---

### 介绍

偶然间看到一篇写于15年的文章，说实话，标题确实吸引了我，不过看了几遍之后，确实精彩。 关于这篇文章，我就不直接翻译了。 项目的需求就是 客户端发送请求，服务端接收请求处理数据(原文是把资源上传至 Amazon S3 资源中)。本质上就是这样,
![image](../../../static/image/handle-million.png)

我稍微改动了原文的业务代码，但是并不影响核心模块。在第一版中，每收到一个 Request,开启一个 G 进行处理，快速响应，很常规的操作。


代码如下

### 初版

```go

package main

import (
	"fmt"
	"log"
	"net/http"
	"time"
)

type Payload struct {
	// 传啥不重要
}

func (p *Payload) UpdateToS3() error {
	//存储逻辑,模拟操作耗时
	time.Sleep(500 * time.Millisecond)
	fmt.Println("上传成功")
	return nil
}

func payloadHandler(w http.ResponseWriter, r *http.Request) {
	// 业务过滤
	// 请求body解析......
	var p Payload
	go p.UpdateToS3()
	w.Write([]byte("操作成功"))
}

func main() {
	http.HandleFunc("/payload", payloadHandler)
	log.Fatal(http.ListenAndServe(":8099", nil))
}

```

这样操作存在什么问题呢？一般情况下，没什么问题。但是如果是高并发的场景下，不对G数进行控制，你的 CPU 使用率暴涨，内存占用暴涨，直至程序奔溃。

如果此操作落地至数据库，例如mysql,那么相应的，你数据库的服务器磁盘IO、网络带宽
、CPU负载、内存消耗都会达到非常高的情况，一并奔溃。所以，一旦程序中出现不可控的事物，往往是危险的信号。


### 中版

```go
package main

import (
	"fmt"
	"log"
	"net/http"
	"time"
)

const MaxQueue = 400

var Queue chan Payload

func init() {
	Queue = make(chan Payload, MaxQueue)
}

type Payload struct {
	// 传啥不重要
}

func (p *Payload) UpdateToS3() error {
	//存储逻辑,模拟操作耗时
	time.Sleep(500 * time.Millisecond)
	fmt.Println("上传成功")
	return nil
}

func payloadHandler(w http.ResponseWriter, r *http.Request) {
	// 业务过滤
	// 请求body解析......
	var p Payload
	//go p.UpdateToS3()
	Queue <- p
	w.Write([]byte("操作成功"))
}

// 处理任务
func StartProcessor() {
	for {
		select {
		case payload := <-Queue:
			payload.UpdateToS3()
		}
	}
}

func main() {
	http.HandleFunc("/payload", payloadHandler)
	//单独开一个g接收与处理任务
	go StartProcessor()
	log.Fatal(http.ListenAndServe(":8099", nil))
}
```

这一版借助带 buffered 的 channel 来完成这个功能，这样控制住了无限制的G，但是依然没有解决问题。 

处理请求是一个同步的操作，每次只会处理一个任务，然而高并发下请求进来的速度远远超过了处理的速度。这种情况，一旦 channel
满了之后， 后续的请求将会被阻塞等地啊。然后你会发现，响应的时间会大幅度的开始增加， 甚至不再有任何的响应。

### 终版

```go
package main

package main

import (
"fmt"
"log"
"net/http"
"time"
)

const (
	MaxWorker = 100 //随便设置值
	MaxQueue  = 200 // 随便设置值
)

// 一个可以发送工作请求的缓冲 channel
var JobQueue chan Job

func init() {
	JobQueue = make(chan Job, MaxQueue)
}

type Payload struct{}

type Job struct {
	PayLoad Payload
}

type Worker struct {
	WorkerPool chan chan Job
	JobChannel chan Job
	quit       chan bool
}

func NewWorker(workerPool chan chan Job) Worker {
	return Worker{
		WorkerPool: workerPool,
		JobChannel: make(chan Job),
		quit:       make(chan bool),
	}
}

// Start 方法开启一个 worker 循环，监听退出 channel，可按需停止这个循环
func (w Worker) Start() {
	go func() {
		for {
			// 将当前的 worker 注册到 worker 队列中
			w.WorkerPool <- w.JobChannel
			select {
			case job := <-w.JobChannel:
				// 	真正业务的地方
				//	模拟操作耗时
				time.Sleep(500 * time.Millisecond)
				fmt.Printf("上传成功:%v\n", job)
			case <-w.quit:
				return
			}
		}
	}()
}

func (w Worker) stop() {
	go func() {
		w.quit <- true
	}()
}

// 初始化操作

type Dispatcher struct {
	// 注册到 dispatcher 的 worker channel 池
	WorkerPool chan chan Job
}

func NewDispatcher(maxWorkers int) *Dispatcher {
	pool := make(chan chan Job, maxWorkers)
	return &Dispatcher{WorkerPool: pool}
}

func (d *Dispatcher) Run() {
	// 开始运行 n 个 worker
	for i := 0; i < MaxWorker; i++ {
		worker := NewWorker(d.WorkerPool)
		worker.Start()
	}
	go d.dispatch()
}

func (d *Dispatcher) dispatch() {
	for {
		select {
		case job := <-JobQueue:
			go func(job Job) {
				// 尝试获取一个可用的 worker job channel，阻塞直到有可用的 worker
				jobChannel := <-d.WorkerPool
				// 分发任务到 worker job channel 中
				jobChannel <- job
			}(job)
		}
	}
}

// 接收请求，把任务筛入JobQueue。
func payloadHandler(w http.ResponseWriter, r *http.Request) {
	work := Job{PayLoad: Payload{}}
	JobQueue <- work
	_, _ = w.Write([]byte("操作成功"))
}

func main() {
	// 通过调度器创建worker，监听来自 JobQueue的任务
	d := NewDispatcher(MaxWorker)
	d.Run()
	http.HandleFunc("/payload", payloadHandler)
	log.Fatal(http.ListenAndServe(":8099", nil))
}


```

最终采用的是两级 channel，一级作为任务的队列。另一级用来控制并发处理任务的 worker 数量。
```go
func payloadHandler(w http.ResponseWriter, r *http.Request) {
	work := Job{PayLoad: Payload{}}
	JobQueue <- work
	_, _ = w.Write([]byte("操作成功"))
}
```
首先我们再接收到一个请求，创建任务 work，把它放入到任务队列中等待 work 池处理。

```go
func (d *Dispatcher) Run() {
	// 开始运行 n 个 worker
	for i := 0; i < MaxWorker; i++ {
		worker := NewWorker(d.WorkerPool)
		worker.Start()
	}
	go d.dispatch()
}

func (d *Dispatcher) dispatch() {
	for {
		select {
		case job := <-JobQueue:
			go func(job Job) {
				// 尝试获取一个可用的 worker job channel，阻塞直到有可用的 worker
				jobChannel := <-d.WorkerPool
				// 分发任务到 worker job channel 中
				jobChannel <- job
			}(job)
		}
	}
}
```

调度器初始化work池后，在 dispatch 中，一旦我们接收到 JobQueue 的任务，就去尝试获取一个可用的 worker，分发任务给work 的 job channel 中。
注意这个过程不是同步的，而是每接收到一个 job,就开启一个 g 去处理。可以保证 JobQueue 不需要进行阻塞，对应的往 JobQueue 理论上也可以不需要阻塞地写入任务。

但是这里"不可控"的G和上面还是不同的，这里G存活数量理论是
```go
G(live)=总请求数-已处理完的G数
```
而且这里的 G 本质上没有像一开始那样，涉及到外部资源、数据库、或者网络带宽的占用。
这里只有当有空闲的 worker 被唤醒，然后分发任务，生命周期远远短于上面的操作。





### 最终方案