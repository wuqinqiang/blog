---
title: "无限缓冲的channel(1)"
date: 2021-04-25T22:25:52+08:00 
toc: true 
tags:

- go
- enum

---

事情的起因是前几周看到鸟窝的一篇关于实现无限缓冲的 channel 的文章，但是那时候忙着别的事，所以也一直没看。

正好今天来瞧瞧，不过这篇文章不会涉及到鸟窝自己实现的 chanx，我们会在第二篇的时候提到。

我们都知道，可以创建无缓冲和有缓冲的 channel。一旦创建的时候指定了 channel 的容量，那么在这个 channel 的生命周期内，我们将无法改变它的容量。

有时候，我们并不知道也无法预估塞入通道的数量规模，并且此时通道的写入速度远远超过通道的消费速度，那么必然会在某个时间点通道塞满，导致发送方阻塞。 此时有人就会提到，能不能提供一个无限缓冲的 channel。

这个问题早在 2017 年就有人提过 issues，最终 go 官方没有实现这个提案。
![image](https://image.syst.top/image/unlimited/unlimit-1.png)

不过，这个 issues 下面总共产生了 67 个 comments，评论很精彩。

比如有人提到:

```
cznic:Unlimited capacity channels ask for a machine with unlimited memory.
rsc:The limited capacity of channels is an important source of backpressure in a set of communicating goroutines. It is typically a mistake to use an unbounded channel, because you lose that backpressure. If one goroutine falls sufficiently behind, you usually want to take some action in response, not just queue its messages forever. The appropriate response varies by situation: maybe you want to drop messages, maybe you want to keep summary messages, maybe you want to take different responses as the goroutine falls further and further behind. Making it trivial to reach for unbounded channels keeps developers from thinking about this, which I believe is a strong disadvantage.
```
那么如果实现一个无限缓冲的 channel 呢？

无非就是在写入通道和从通道读取数据之间设置一个中间层。

针对这类需求，有很多版本的实现，我们来看其中的一个实现。鸟窝的 chanx 就是在这个基础上做修改的。


