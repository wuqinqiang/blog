---
title: "channel原理解析"
date: 2021-04-25T22:25:52+08:00 
toc: true 
tags:

- go
- enum

---



躺的太久了，不能太躺了。

宁可我卷死别人，不能让别人卷我。

之前断断续续看过`Go`几个模块的源码，可从未下笔，导致有些细节记不起来了。打算写一些文章重新记录。

`channel`源码解析文章太多了，很多上来都是直接讲解源码的，对于完全没看过源码的人来说，就挺突然的。鉴于长篇大论大部分人没耐心看完，我打算分两三篇彻底讲清楚`channel`，最后附上完整的`ppt`。

当然这当中不会涉及过多细节源码，有时候，细节是魔鬼。



### 介绍

channel一些基础介绍这里就不过多涉及了，我不相信都1202年了，用过`Go`的人没用过`channel`。

当然下图也涵盖了大部分使用方法。注意是使用姿势，而不是模式。

![image](https://cdn.syst.top/use-channel.png)



顺便再提一句，有一道使用`channel`进行任务编排的经典的题。题目如下，

有四个`goroutine`，编号为 1、2、3、4。每秒钟会有一个 `goroutine`打印自己的编号。请你实现这个程序，让输出的编号总是按照 1、2、3、4、1、2、3、4、……的顺序打印出来。可以自己先思考下，答案也可以通过后台回复x x x获取。



### 原理解析

从一个简单的例子说起。创建一个`main.go`文件，代码如下，

![image](https://cdn.syst.top/ch-send.png)

当然，你直接执行这段代码会显示死锁，因为系统会检测到没有其他`g`有接收消息的动作。

我们来看看这段代码编译以后长啥样。

得到`go`程序的汇编代码并不难。

可以使用`go tool compile -N -l -S main.go`生成汇编代码：

![image](https://cdn.syst.top/compile-channel.png)



或者使用`go tool compile -N -l main.go` 先编译出代码，然后再使用`go tool objdump main.o`反汇编出代码。 

当然还可以使用`go build -gcflags -S main.go`也可以得到汇编的代码。

上面两种我就不演示了，可以自行实验。他们之中的`flag`具体含义也可以自行了解，这些都不是本章的重点。



如果你觉得上面要自己敲代码比较麻烦，我推荐一个更加直接的可视化的工具。

![image](https://cdn.syst.top/compile-show.png)



综上，从编译的代码我们可以看出，上述初始化一个`channel`,

```go
	ch := make(chan struct{})
```

实际上调用的是`runtime.makechan`。

我们来了解一下`runtime.makechan`

![image](https://cdn.syst.top/makechannel.png)

从函数中，我们能知道最终返回一个`runtime.hchan`的指针。

`runtime.hchan`结构。

![image](https://cdn.syst.top/hchan.png)



我们先来解释`hchan`结构体各个字段的含义，之后在案例介绍中会更加详细的说明他们的作用。

![image](https://cdn.syst.top/hchan-detail2.png)

我们先来看`qcount`和`dataqsiz`有什么区别？

你去银行办事，银行有5个办事窗口，那么`dataqsiz`就等于5。在这里体现的是`channel`的容量为5。去银行的时候，当前有3个窗口有人正在办事，那么`qcount`就等于3，体现`channel`当前有3个数据元素。那么此时银行还可以再接待2个客户，对应还可以往`channel`发送2个数据元素。



其他字段现在看看说明就行了，后面会细讲。

到这里我们就知道创建一个`channel`本质上就是得到一个`runtime.hchan`的指针，后续对此`chan`的操作，无非就是对结构体字段进行相对应的操作。

同时我们也能猜出，为啥`channel`能在不同的g中传递消息而对于使用者来说不用担心并发的问题，本质上就是`hchan`内部使用互斥锁来保证了并发安全。

最后我们来看一下`runtime.makechan`函数核心实现，当然注释已经很明白了。

![image-20211009084700993](https://cdn.syst.top/makechan2.png)

可以看到创建的时候有一段`switch`分支代码，那么什么情况下会走对应的`case`呢?

![image-20211009084700993](https://cdn.syst.top/make-zi.png)

根据上面的信息，我们可以得出，

- 如果创建一个无缓冲`channel` ，那么只需要为`runtime.hchan`本身分配一段内存空间即可。
- 如果创建的缓冲`channel` 存储的类型不是指针类型，会为当前 `channel` 和存储类型元素的缓冲区分配一块连续的内存空间。
- 在默认情况下(即存储类型包含指针)，会单独为`runtime.hchan`和缓冲区分配内存；



### 总结

这篇我们主要介绍了如何获取`go`程序的汇编代码，通过汇编代码知道创建`channel`的具体函数`runtime.makechan`，同时我们还知道不同的创建姿势会导致走向不同的内存空间分配。最后通过创建函数我们知道`channel`在程序运行时使用`runtime.hchan`来表示。

下一篇我们继续。

