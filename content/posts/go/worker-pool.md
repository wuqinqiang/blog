---
title: "go并发-工作池模式"
date: 2021-07-01T23:37:45+08:00
toc: true
tags :

- go
- test

---

### 开篇
之前写过一篇文章，它有个响亮的名字: `Handling 1 Million Requests per Minute with Go`。
这是国外的一个作者写的，我做了一篇说明。起的也是这个标题，
阅读量是我最好的一篇，果然文章都是靠标题出彩的.....

今天偶然看到另一篇文章(原文在文末)。两篇文章原理相似:有一批工作任务(job)，通过工作池(worker-pool)的方式，达到多 `worker` 并发处理 `job` 的效果。

他们还是有很多不同的点，实现上差别也是蛮大的。

首先上一篇文章我放了一张图片，大概就是上篇整体的工作流。
![image](https://image.syst.top/image/work-pool.png)

- 每个 `worker` 处理完任务就好，不关心结果,不对结果做进一步处理。
- 只要请求不停止，程序就不会停止，没有控制机制，除非宕机。

这篇文章不同点在于:

首先数据会从 `generate` (生产数据)->并发处理数据->处理结果聚合。
图大概是这样的,
![image](https://image.syst.top/image/out.png)

然后它可以通过 `context.context` 达到控制工作池停止工作的效果。

最后通过代码，你会发现它不是传统意义上的 `worker-pool`，后面会说明。


下图能清晰表达整体流程了。
![image](https://image.syst.top/image/work-pool-2.png)

顺便说一句，这篇文章实现的代码比 `Handling 1 Million Requests per Minute with Go` 的代码简单多了。


首先看 `job`。

```go
package wpool

import (
	"context"
)

type JobID string
type jobType string
type jobMetadata map[string]interface{}

type ExecutionFn func(ctx context.Context, args interface{}) (interface{}, error)

type JobDescriptor struct {
	ID       JobID 
	JType    jobType
	Metadata map[string]interface{}
}

type Result struct {
	Value      interface{}
	Err        error
	Descriptor JobDescriptor
}

type Job struct {
	Descriptor JobDescriptor
	ExecFn     ExecutionFn
	Args       interface{}
}

// 处理 job 逻辑,处理结果包装成 Result 结果
func (j Job) execute(ctx context.Context) Result {
	value, err := j.ExecFn(ctx, j.Args)
	if err != nil {
		return Result{
			Err:        err,
			Descriptor: j.Descriptor,
		}
	}

	return Result{
		Value:      value,
		Descriptor: j.Descriptor,
	}
}
```
这个可以简单过一下。最终每个 `job` 处理完都会包装成 `Result` 返回。

下面这段就是核心代码了。

```go
package wpool

import (
	"context"
	"fmt"
	"sync"
)

// 运行中的每个worker
func worker(ctx context.Context, wg *sync.WaitGroup, jobs <-chan Job, results chan<- Result) {
	defer wg.Done()
	for {
		select {
		case job, ok := <-jobs:
			if !ok {
				return
			}
			results <- job.execute(ctx)
		case <-ctx.Done():
			fmt.Printf("cancelled worker. Error detail: %v\n", ctx.Err())
			results <- Result{
				Err: ctx.Err(),
			}
			return
		}
	}
}

type WorkerPool struct {
	workersCount int //worker 数量
	jobs         chan Job // 存储 job 的 channel 
	results      chan Result // 处理完每个 job 对应的 结果集
	Done         chan struct{} //是否结束
}

func New(wcount int) WorkerPool {
	return WorkerPool{
		workersCount: wcount,
		jobs:         make(chan Job, wcount),
		results:      make(chan Result, wcount),
		Done:         make(chan struct{}),
	}
}

func (wp WorkerPool) Run(ctx context.Context) {
	var wg sync.WaitGroup
	for i := 0; i < wp.workersCount; i++ {
		wg.Add(1)
		go worker(ctx, &wg, wp.jobs, wp.results)
	}

	wg.Wait()
	close(wp.Done)
	close(wp.results)
}

func (wp WorkerPool) Results() <-chan Result {
	return wp.results
}

func (wp WorkerPool) GenerateFrom(jobsBulk []Job) {
	for i, _ := range jobsBulk {
		wp.jobs <- jobsBulk[i]
	}
	close(wp.jobs)
}

```

整个 `WorkerPool` 结构很简单。 `jobs` 是一个缓冲 `channel`。每一个任务都会放入 `jobs` 中等待处理 `woker` 处理。

`results` 也是一个通道类型，它的作用是保存每个 `job` 处理后产生的结果 `Result`。

首先通过 `New` 初始化一个 `worker-pool` 工作池,然后执行 `Run` 开始运行。

```go
func New(wcount int) WorkerPool {
	return WorkerPool{
		workersCount: wcount,
		jobs:         make(chan Job, wcount),
		results:      make(chan Result, wcount),
		Done:         make(chan struct{}),
	}
}
func (wp WorkerPool) Run(ctx context.Context) {
	var wg sync.WaitGroup

	for i := 0; i < wp.workersCount; i++ {
		wg.Add(1)
		go worker(ctx, &wg, wp.jobs, wp.results)
	}

	wg.Wait()
	close(wp.Done)
	close(wp.results)
}
```

初始化的时候传入 `worker` 数，对应每个 `g` 运行 `work(ctx,&wg,wp.jobs,wp.results)`,组成了 `worker-pool`。
同时通过 `sync.WaitGroup`,我们可以等待所有 `worker` 工作结束，也就意味着 `work-pool` 结束工作，当然可能是因为任务处理结束，也可能是被停止了。

每个 `job` 数据源是如何来的？

```go
// job数据源，把每个 job 放入到 jobs channel 中
func (wp WorkerPool) GenerateFrom(jobsBulk []Job) {
	for i, _ := range jobsBulk {
		wp.jobs <- jobsBulk[i]
	}
	close(wp.jobs)
}
```

对应每个 `worker` 的工作，

```go
func worker(ctx context.Context, wg *sync.WaitGroup, jobs <-chan Job, results chan<- Result) {
	defer wg.Done()
	for {
		select {
		case job, ok := <-jobs:
			if !ok {
				return
			}
			results <- job.execute(ctx)
		case <-ctx.Done():
			fmt.Printf("cancelled worker. Error detail: %v\n", ctx.Err())
			results <- Result{
				Err: ctx.Err(),
			}
			return
		}
	}
}
```

每个 worker 都尝试从同一个 `jobs` 获取数据，这是一个典型的 `fan-out` 模式。
当对应的 `g` 获取到 `job` 进行处理后，会把处理结果发送到同一个 `results channel` 中,这又是一个 `fan-in` 模式。
当然我们通过 `context.Context` 可以对每个 `worker` 做停止运行控制。

最后是处理结果集合，
```go
// 处理结果集
func (wp WorkerPool) Results() <-chan Result {
	return wp.results
}
```

那么整体的测试代码就是:

```go
func TestWorkerPool(t *testing.T) {
	wp := New(workerCount)

	ctx, cancel := context.WithCancel(context.TODO())
	defer cancel()

	go wp.GenerateFrom(testJobs())

	go wp.Run(ctx)

	for {
		select {
		case r, ok := <-wp.Results():
			if !ok {
				continue
			}

			i, err := strconv.ParseInt(string(r.Descriptor.ID), 10, 64)
			if err != nil {
				t.Fatalf("unexpected error: %v", err)
			}

			val := r.Value.(int)
			if val != int(i)*2 {
				t.Fatalf("wrong value %v; expected %v", val, int(i)*2)
			}
		case <-wp.Done:
			return
		default:
		}
	}
}
```

看了代码之后，我们知道，这并不是一个传统意义的 `worker-pool`。它并不像 `Handling 1 Million Requests per Minute with Go` 这篇文章一样，
初始化一个真正的 `worker-pool`，一旦接收到 `job`,就尝试从池中获取一个 `worker`，
把对应的 `job` 交给这个 `work` 进行处理，等 `work` 处理完毕，重新进行到工作池中，等待下一次被利用。





















 






















