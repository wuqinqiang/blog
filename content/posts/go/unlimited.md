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

我们都知道，channel 有两种类型:无缓冲和有缓冲的。一旦创建的时候指定了 channel 的类型(即容量)，那么在这个 channel 的生命周期内，我们将无法改变它的容量。

有时候，我们并不知道也无法预估塞入通道的数量规模，并且通道的写入速度远远超过读取速度，那么必然会在某个时间点通道塞满，导致写入阻塞。
此时有人就会提到，能不能提供一个无限缓冲(unbounded or Unlimited)的 通道。

这个问题早在 2017 年就有人提过 issues，最终 go 官方没有实现这个提案。

不过，这个 issues 下面总共产生了 67 个 comments，评论很精彩。
![image](https://image.syst.top/image/unlimited/unlimit-1.png)

比如有人提到:

```
cznic:Unlimited capacity channels ask for a machine with unlimited memory.
rsc:The limited capacity of channels is an important source of backpressure in a set of communicating goroutines. It is typically a mistake to use an unbounded channel, because you lose that backpressure. If one goroutine falls sufficiently behind, you usually want to take some action in response, not just queue its messages forever. The appropriate response varies by situation: maybe you want to drop messages, maybe you want to keep summary messages, maybe you want to take different responses as the goroutine falls further and further behind. Making it trivial to reach for unbounded channels keeps developers from thinking about this, which I believe is a strong disadvantage.
```
那么如何实现一个无限缓冲的 channel 呢？

针对这类需求，有很多版本的实现，我们来看其中的一个实现。鸟窝的 chanx 就是在这个基础上做修改的。

我们一步步还原它的实现，这其中还能知道作者的思考过程。

第一版,

![image](https://image.syst.top/image/unlimited/unlimit-2.png)

然后有个 test。

![image](https://image.syst.top/image/unlimited/unlimit-3.png)

![image](https://image.syst.top/image/unlimited/unlimit-4.png)

当走到第二个 case 的时候，由于 inQueue 一开始是空的，那么必然会出现  index out。
不仅是一开始，在运行中，如果读取比写入快，那么必然也会导致相同的情况。

改了一版，
![image](https://image.syst.top/image/unlimited/unlimit-5.jpg)
![image](https://image.syst.top/image/unlimited/unlimit-6.png)

在 inQueue 没有值的时候，我们把 nil 也写入到通道，
然后测试代码中我们从 out channel 读取数值试图传唤成 int 失败了。
当队列中没有数据时，我们不应该写入 out 通道。

![image](https://image.syst.top/image/unlimited/unlimit-7.jpg)
作者使用了一个技巧，如果 inQueue 没有数据，那么尝试写入一个 nil 通道将永远阻塞。
通常，永久阻塞是一个不好的行为，但是这个是包含在 select 语句中的，所以问题不大。

![image](https://image.syst.top/image/unlimited/unlimit-8.png)
还是有问题，原因很简单，我们再发送完数据就马上关闭了 in 通道。随后 break loop。接下来关闭 out 通道，程序运行结束。 
inQueue 还有值未被读取。

只要写比多快，那么就永远存在这个问题。我们需要保证在通道关闭的时候，inQueue 已为空。
![image](https://image.syst.top/image/unlimited/unlimit-9.png)

