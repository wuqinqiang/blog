---
title: "无限缓冲的channel(2)"
date: 2021-04-25T22:25:52+08:00 
toc: true 
tags:

- go
- enum

---
## chanx
上篇文章我们提到，当我们创建一个有缓冲的通道并指定了容量，那么在这个通道的生命周期内，我们将再也无法改变它的容量。
容量是有限的。在典型的写比读快的场景，容易导致通道被塞满，由此引发了关于无限缓存的 channel 讨论。
最终我们分析了一个实现的方案。
最后，我们也提到了此方案还可以优化的点。

鸟窝的 chanx 正是基于此方案改造而成的。

上篇文章说过，所谓的无限缓冲，无非是借助一个中间层的数据结构，暂存数据。

在 chanx 中是这样的:
```go
type UnboundedChan struct {
	In     chan<- T    // channel for write
	Out    <-chan T    // channel for read
	buffer *RingBuffer // buffer
}
```

`in` 和 `out` 的职责在上篇文章已经说明，这里的 `buffer` 就是我们所谓的中间层。这里的 `RingBuffer` 我们后面再说。

```go
func NewUnboundedChan(initCapacity int) UnboundedChan {
	return NewUnboundedChanSize(initCapacity, initCapacity, initCapacity)
}

func NewUnboundedChanSize(initInCapacity, initOutCapacity, initBufCapacity int) UnboundedChan {
	in := make(chan T, initInCapacity)
	out := make(chan T, initOutCapacity)
	ch := UnboundedChan{In: in, Out: out, buffer: NewRingBuffer(initBufCapacity)}

	go process(in, out, ch)

	return ch
}
```

提供了两个 初始化 `UnboundedChan` 的方法，从代码中我们可以明显的看出,`NewUnboundedChanSize` 可以给每个属性自定义自己的容量大小。仅此而已。

这里就有和上篇文章的不同之处了。
![image](https://image.syst.top/image/unlimited/unlimit-2.png)

可以清楚得看到，上篇的 `in` 和 `out` 是一个无缓冲的通道，而这里是两个缓冲通道，这和他们对数据的流转处理有很大关系。

我们接下去看 `process(in,out,ch)` 最核心的方法。
![image](https://image.syst.top/image/unlimited-2/unlimit-1.png)

这时候，我们再放上上一篇的核心操作代码,
![image](https://image.syst.top/image/unlimited/unlimit-7.jpg)

可以很明显他们两的区别。
上篇从 in 取数据会先塞入到 buffer，然后从 buffer 中取数据赛到 out。
而 chanx 从 in 取出数据后先直接塞到 out(没有中间商赚差价)，只有在 out 已经满的情况下，才塞入到 buffer。

chanx 还有一段小细节代码。
![image](https://image.syst.top/image/unlimited-2/unlimit-2.jpg)

能走到这里，一定是因为 out 满了。我们在把值塞入到 buffer 的同时，需要尝试把 buffer 中的数据塞入到 out 。此时 in 并没有关闭，还在持续的写入数据，
为了避免 in 塞满，我们还需要尝试从 in 中取数据到 buffer。

## buffer

上篇文章我提到了关于 buffer 优化的点。chanx 是如何优化的。

```go
// type T interface{}

type RingBuffer struct {
	buf         []T 
	initialSize int
	size        int
	r           int // read pointer
	w           int // write pointer
}
```
这是 `buffer` 的结构，其中
-  buf 具体存储数据的结构。
- initialSize 舒适化 buf 的长度
- size 当前 buf 的长度
- r 当前读数据位置
- w 当前写入数据位置

本质上就是一个环形的缓冲区，达到资源的复用，当 buffer 满了，提供自动扩容的功能。
我们来看一个写入操作。
![image](https://image.syst.top/image/unlimited-2/unlimit-3.png)

接着看扩容。
![image](https://image.syst.top/image/unlimited-2/unlimit-4.png)

这段代码唯一难理解的就是数据迁移了。这里的数据迁移目的是为了保证先入先出的原则。
可能加了注释有些人也无法理解，那么就再加一个草率图。
假设我们 buffer 的长度是 8。 当前读和写的 index 都是5。
![image](https://image.syst.top/image/unlimited-2/unlimit-5.png)

当 buffer 为空并且当前的 `size` 比初始化 `size` 还大，那么可以考虑重置 `buffer` 了。
```go
//if ch.buffer.IsEmpty() && ch.buffer.size > ch.buffer.initialSize { 
//						ch.buffer.Reset()
//					}
func (r *RingBuffer) Reset() {
r.r = 0
r.w = 0
r.size = r.initialSize
r.buf = make([]T, r.initialSize)
}
```
剩下的代码就没什么好说的了。

### 总结
继上篇文章后，这篇文章我们主要讲解了 chanx 是如何实现无限缓冲的 channel 的。
其中最重要的一个点在于 chanx 中 buffer 的实现采用的是 `ringbuffer`，达到资源的复用的同时还能自动扩容。



