---
title: "hystrix-go使用与原理(二)"
date: 2021-04-21T23:54:52+08:00
toc: true
tags :
- go
---

### 开篇
上篇文章主要介绍了 `hystrix-go` 的使用以及原理，这篇文章让我们全面的解析源码。本文很长，请耐心看完。另外，由于直接放源码很是影响手机阅读体验，我把源码都截成图片了。

还是上篇的例子

![](https://cdn.learnku.com/uploads/images/202012/29/26855/kfaOfFI9ez.png!large)
大部分文章只是说明每个模块的职责和功能，考虑到如果只是单纯说明，读者还是很难把整体的流程连接起来。因此我打算从这个例子一步步解析。
就直接从开头 `hystrix.ConfigureCommand` 开始吧。
![](https://cdn.learnku.com/uploads/images/202012/29/26855/Twv8QvPbyR.png!large)

上篇文章提过，这个操作主要是为每个 `commandName` 自定义自己的规则配置。如果未自定义，那么会使用默认值。 最终会把配置值存储在 `circuitSettings` 这个 `map`类型中，它的初始化操作是在 `init()`执行的。
![](https://cdn.learnku.com/uploads/images/202012/29/26855/4j1xoqVqzB.png!large)
接下来执行 `hystrix.Do`，	`Do` 是一个同步的操作，它会阻塞等待，直到执行函数结束或者熔断器返回错误，如:断路器开启、超出最大并发数。
![](https://cdn.learnku.com/uploads/images/202012/29/26855/ENUcELdfsT.png!large)
此函数需要三个参数，第一个参数表示 `commandName` 的名称，第二个参数就是正常的业务的匿名函数，比如在函数中进行外部服务调用。如果调用失败，那么就会执行第三个参数的操作，我们可以称之为保底操作，当熔断器开启的时候，系统也是会直接调用此函数。这两个参数的类型分别是匿名函数和闭包函数。
![](https://cdn.learnku.com/uploads/images/202012/29/26855/Vn1Yot5vsx.png!large)
从图中可以看出，`Do` 函数只是把传入的后两个参数进一步封装成函数。然后调用 `DoC`。

![](https://cdn.learnku.com/uploads/images/202012/29/26855/eJ89Y8YEHw.png!large)

`Doc` 函数第一个参数是上下文 `context.Context`。`context.Context` 一般出现在不同 `Goroutine` 之间同步指定数据。如果你使用过 `gin` 框架，经常和它打交道。

第三和第四的参数即 `Do` 中进一步包装的两个闭包函数。所谓闭包，我的理解是:存在自由的变量。这个自由的变量取决于运行闭包函数时的环境，在 `DoC`中,` runFunc` 和 `fallbackFuncC` 类型
![hystrix-go 源码解析](https://cdn.learnku.com/uploads/images/202012/29/26855/12x0zgb6ll.png!large)
也就是说这两个闭包的自由变量是 `context.Context`。

`Doc`函数中 变量 `r` 和 `f` 不再解释。由于我们在调用 `Do`函数时传递了第三个参数,因此执行 `errChan = GoC(ctx, name, r, f)`。最下面使用`select` 可以监控多 `channel`。当某个  `channel` 有数据时，从其中读取。我们接着往下看 `GoC`。

`Goc`是核心代码块。它是真正执行你的核心业务函数的地方。我先大体介绍一些这个函数核心功能。
它会先去验证一些规则，比如判断熔断器是否开启，决定当前是否可以执行你的业务。判断是否可以获取访问令牌。如果可以，执行你的业务逻辑，成功了上报成功的状态，失败了，除了上报状态，如果传入了异常处理的函数，那么执行异常处理的函数。另外还有一些归还令牌等操作。我们来看代码。
![](https://cdn.learnku.com/uploads/images/202012/29/26855/2zG2xABgMO.png!large)
这段代码就不解释了吧。但是我们可以来看看 `command` 结构体中还有啥参数。
![](https://cdn.learnku.com/uploads/images/202012/29/26855/fpbWU6cyKp.png!large)
`command` 里面，关键的两个字段是 `events` 和 `circuit`。其实 `events` 主要是存储事件类型信息，比如执行成功的 `success`，或者失败的 `timeout`、`context_canceled` 等。 circuit 是指针类型 `CircuitBreaker`, `CircuitBreaker`就是真正的熔断器 。`command` 主要是记录单个执行的状态以及和熔断器进行一些运行交互。交互什么? 主要向 `CircuitBreaker` 上报执行状态事件。

接下来看下面的代码
![](https://cdn.learnku.com/uploads/images/202012/29/26855/vwYVVDHUsQ.png!large)
`GetCircuit(name)`，从函数名就知道是通过名称获取一个熔断器。
![](https://cdn.learnku.com/uploads/images/202012/30/26855/fgMesgzwlm.png!large)
这个函数代码很清晰，如果没有从 `circuitBreakers`中查询到对应的 `CircuitBreaker`，那么就创建一个。

这里的代码有点小细节。首先 `circuitBreakers`是 `map` 类型，我们都知道 `map` 并不是并发安全的。所以在查找的的时候加了读锁。如果没找到值，那么解除读锁。尝试获取写锁，我们在写锁里面，又进一步确认是否存在`circuitBreakers`，为什么需要这样操作? 这是因为在我们释放读锁到获取写锁过程中有可能存在其他的 `Goroutine` 抢先一步创建。所以这里需要进一步确认，此时不存在，那就真的不存在，通过 `name` 生成一个 `circuitBreakers`。具体看下 `newCircuitBreaker(name)` 函数。
![](https://cdn.learnku.com/uploads/images/202012/30/26855/aWyPBGteD6.png!large)
主要是初始化创建一个熔断器 `CircuitBreaker`操作，我们可以看看 `CircuitBreaker` 都有哪些字段。

![一文读懂 hystrix-go 源码](https://cdn.learnku.com/uploads/images/202012/30/26855/frqN8a2tWj.png!large)
主要说明几个字段，`open` 表示当前熔断器是否开启，`executorPool` 是流量控制中心，所有的请求都需要先获取到令牌。`metrics` 的类型是 `*metricExchange`,可以看成是上报执行状态事件的载体。通过它把执行状态信息存储到实际熔断器执行各个维度状态(成功次数，失败次数，超时......)的数据集合中。

`newCircuitBreaker(name)` 初始化的同时也初始化了 `executorPool` 和 `metrics`。

先看 `newMetricExchange(name)`。看看它 `metricExchange` 结构
![](https://cdn.learnku.com/uploads/images/202012/30/26855/VkJb7VQ41k.png!large)
`Updates` 是一个 `channel` 类型，通过 `Updates` 上报执行事件集合。`metricCollectors` 存储的是 `metricCollector.MetricCollector` 切片，而 `metricCollector.MetricCollector` 是一个接口类型。

![一文读懂 hystrix-go 源码](https://cdn.learnku.com/uploads/images/202012/30/26855/36K9RaCfiC.png!large)

`newMetricExchange(name)` 中，
![](https://cdn.learnku.com/uploads/images/202012/30/26855/QMt69SGi7l.png!large)
可以看到，初始化 `Updates` 通道的的容量是 `2000`。初始化 `metricCollectors` 主要逻辑在 `InitializeMetricCollectors`。
![](https://cdn.learnku.com/uploads/images/202012/30/26855/2J1yday7lA.png!large)
关键的地方我标明了。再看看 `newDefaultMetricCollector`
![](https://cdn.learnku.com/uploads/images/202012/30/26855/JWj5RbQxEf.png!large)
此函数返回一个 `MetricCollector` 类型，结构体 `DefaultMetricCollector` 实现了 `MetricCollector` 所有方法。再看看 `DefaultMetricCollector`，不正是存储熔断器执行状态所有信息嘛。看看指针类型 `rolling.Number`：
![](https://cdn.learnku.com/uploads/images/202012/30/26855/MEeJ15Zk69.png!large)
`rolling.Number` 是真正存储各个执行事件状态信息的底层存储结构。它是如何只保存 `10` 秒内的信息的。
![](https://cdn.learnku.com/uploads/images/202012/30/26855/vkVXeOEHOe.png!large)
惊不惊喜？
`newMetricExchange(name)` 还有一个细节，会单独开启一个 `g` 运行  `go m.Monitor()`  去接收 `channel` 类型的 `Updates` 信息，即执行事件状态信息。
![](https://cdn.learnku.com/uploads/images/202012/30/26855/oNJhRa2nEr.png!large)
在接收到事件信息后，调用 `IncrementMetrics` 先做状态信息的整合，最终把整合后的执行状态事件信息上报 `collector.Update(r)`。
![](https://cdn.learnku.com/uploads/images/202012/30/26855/L1CUsHg6NP.png!large)

![](https://cdn.learnku.com/uploads/images/202012/30/26855/iw35u2p7Vg.png!large)

接着回头看初始化流量控制中心 `newExecutorPool(name)`。先看看 `executorPool`结构。
![](https://cdn.learnku.com/uploads/images/202012/30/26855/oJf8DzKrYA.png!large)
主要关注两个字段，`Tickets`表示的就是访问令牌带缓冲通道的 `channel` ，初始化 `channel` 容量取决于一开始你设置的`MaxConcurrentRequests `。当有请求到来时，从 `channel` 中拿出一个令牌，调用后重新归还。 `poolMetrics` 就是流量控制的具体指标。
![](https://cdn.learnku.com/uploads/images/202012/30/26855/Tl4WyBnbah.png!large)

`Executed`表示当前桶已经处理的请求数量。此外在 `newExecutorPool(name)` 函数中，和刚才套路一样，启动一个 `go m.Monitor()`   专门去更新当前桶的最大值。
![](https://cdn.learnku.com/uploads/images/202012/30/26855/UQaHVMIHKw.png!large)
![](https://cdn.learnku.com/uploads/images/202012/30/26855/I1FTB6RksV.png!large)

到这里，`GetCircuit(name)` 获取一个熔断器的代码讲完了。
回到 `GoC` ，我们得到一个 `circuitBreakers` 的指针。
![](https://cdn.learnku.com/uploads/images/202012/30/26855/HCMq26mFlm.png!large)
接下来我们创建一个条件变量 `sync.NewCond`。条件变量的场景是当共享资源发生变化时，通知那些被互斥锁锁住的线程。
在这里它是用来协调通知你可以归还访问令牌了。

接着有一句 `returnOnce := &sync.Once{}`,关于 `sync.Once`,之前解析过一篇文章[你真的了解 sync.Once 吗](https://mp.weixin.qq.com/s?__biz=MzU3Mzc5NTU2MQ==&tempkey=MTA5NF95VUwzVkc5bUNoWkNzZWtwQ1A1dXlPNGZTazVBS3RUZ3FOTDVxRmlGR3M3Tnk0YmJmLURhc1JkQTlkVFhEQ3FxQWRvUXFjeExXS0hBZ2ttVWNFdjR5cGRrUW5HbzJZdTZ2TzkyM2lfT0FIQU9wNzJOVmRHVWNMaTF2eF9odGppb0xVTEU3UzhLRTdieVlEY29NXzliODU3YWJsczhjc3BZblV0bVFRfn4%3D&chksm=7d3d7a344a4af322ddae8dc18b7944c97eca68df27d7f7ff926ee59c89d6b5ba8b5f2de5a8e4#rd "你真的了解 sync.Once 吗")。这里它存在的意义是什么? 我们往下看，就会发现其实到后面开启了两个 `Goroutine`。
![](https://cdn.learnku.com/uploads/images/202012/30/26855/r87TmWEEGW.png!large)
它的作用是确保由最快那个 `Goroutine` 运行 `errWithFallback()` 和 `reportAllEvent()`，而且保证只会执行一次。

接着往下看，
![一文读懂 hystrix-go 源码](https://cdn.learnku.com/uploads/images/202012/30/26855/UMDw3GhSuW.png!large)
这个函数就是就是用来上报执行事件的。
![](https://cdn.learnku.com/uploads/images/202012/30/26855/DlLsJohHro.png!large)

前面都好懂，我们从这段开始看：
![](https://cdn.learnku.com/uploads/images/202012/30/26855/3XV3I8nHPG.png!large)
这里操作这句话是什么意思？因为存在一种情况：当前熔断器是开启的，并且已经过了 `SleepWindow` 时间，此时请求就属于半开的状态，允许尝试执行，如果执行成功，那么就说明服务恢复了，可以关闭熔断器了。接下来，
![](https://cdn.learnku.com/uploads/images/202012/30/26855/xo6FXuJjxA.png!large)
组装执行状态状态事件，然后塞进 `Updates` 通道中。正好被初始化 `metricExchange` 另开的 `Goroutine` 接收，这样，这个上报流程就对应上了。

接下来就是刚才截图的两个 `Goroutine`,我们先看第一个。
![](https://cdn.learnku.com/uploads/images/202012/30/26855/kSrIBnzDZp.png!large)

截图了上前半部分。上来一个 `defer func() { cmd.finished <- true }()` 作为正常运行结束的通知。然后就是 `cmd.circuit.AllowRequest()`判断是否能请求。
![](https://cdn.learnku.com/uploads/images/202012/30/26855/nTLNWvkzEE.png!large)
有两种情况可以往下走，第一种熔断器是关闭的。对应 `circuit.IsOpen()`。
![](https://cdn.learnku.com/uploads/images/202012/30/26855/Mq6gtXmpGu.png!large)
首先判断熔断器是否被强制开启或者已经开启，如果是，直接返回 `true`。否则说明是当前熔断器处于关闭。接着判断过去十秒内各个桶值的和是否小于设置的 `RequestVolumeThreshold `值，如果小于，说明熔断器还应该是关闭状态，返回 `false` 。如果大于等于，那么应该进一步去判断错误百分比 是否超出自己的设置的`ErrorPercentThreshold `。如果超出了，那么说明错误率过高，此时需要开启熔断器。
![](https://cdn.learnku.com/uploads/images/202012/30/26855/tZ1MK9ICVI.png!large)
这段代码很好懂，有意思的是 `int(errPct + 0.5)`。`+0.5` 是为了四舍五入，如果直接浮点数转换成整形，那么必然就是去除小数点。加了 `0.5`，那么假设 `ErrorPercentThreshold`设置 `50`，如果是 `<45.5`,熔断器就继续关闭,否则开启熔断器。

如果`circuit.IsOpen()`不符合，那么再看 `circuit.allowSingleTest()`。虽然熔断器是开启的，但是如果当前的时间已经大于 （上次开启熔断器的时间+`SleepWindow ` 的时间），这时候熔断器属于半开的状态，可以执行下一步。那么就会返回 `true`。

如果 `cmd.circuit.AllowRequest` 返回 `false`，那么就是执行 `returnTicket` 归还令牌(尽管这时候还没有令牌可言)。这段代码很有趣，通过变量  `ticketChecked`  加 `sync.NewCond` 实现的逻辑。`cmd.errorWithFallback()`，上报熔断器已开启事件(`circuit open`)以及运行 `fallBakck` 保底函数(如果存在的话)，执行结束，响应。

![](https://cdn.learnku.com/uploads/images/202012/30/26855/UaVlqg18CJ.png!large)

如果返回 `true`，接着玩下走，

![](https://cdn.learnku.com/uploads/images/202012/30/26855/beu0NFqgrd.png!large)
如果拿不到访问令牌，那么和刚才一样，上报当前请求已超过并发数事件(`max concurrency`)，运行保底操作，响应。

如果拿到访问令牌，那么真正执行自己的业务代码 `run(ctx)`。套路和上面相似。
再看第二个 `Goroutine`
![](https://cdn.learnku.com/uploads/images/202012/30/26855/bMP0fZPdsi.png!large)
 三个 `case`分别表示:正常执行结束、业务执行被取消以及超时。至于为什么说 `Do`是同步操作，因为在 `Doc` 最后,
 ![](https://cdn.learnku.com/uploads/images/202012/30/26855/HBZXgbhOxA.png!large)，而 `Go` 则是一个异步过程。
 到这里，从 `Do` 开始执行的大体流程就走完了。
 最后我们可以直接给出它的大体流程图。
 ![](https://cdn.learnku.com/uploads/images/202012/30/26855/YGB2gucOJF.png!large)
 他们结构体之间的一些关系图。
 ![](https://cdn.learnku.com/uploads/images/202012/30/26855/56cG9Ir6NW.png!large)
 分享 一些奇奇怪怪的东西

![图片](https://cdn.learnku.com/uploads/images/202012/12/26855/d1Y1mxQjr9.webp!large)

👆 长按关注库里的深夜食堂





 





















