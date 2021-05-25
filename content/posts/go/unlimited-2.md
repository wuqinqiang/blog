---
title: "无限缓冲的channel(2)"
date: 2021-04-25T22:25:52+08:00 
toc: true 
tags:

- go
- enum

---
## chanx
上篇文章我们提到，当我们创建一个有缓冲的通道并指定了容量，那么在这个通道的生命周期内，我们将再也无法改变它的容量。 由此引发了关于无限缓存的 `channel` 话题讨论。
我们分析了一个实现无限缓冲的代码。 最后，我们也提到了它还可以继续优化的点。

鸟窝的 `chanx` 正是基于此方案改造而成的，我们来看看他俩的不同之处。

上篇文章说过，所谓的无限缓冲，无非是借助一个中间层的数据结构，暂存临时数据。

在 `chanx` 中，结构是这样的:
```go
type UnboundedChan struct {
	In     chan<- T    // channel for write
	Out    <-chan T    // channel for read
	buffer *RingBuffer // buffer
}
```

`in` 和 `out` 的职责在上篇文章已经说明，这里的 `buffer` 就是我们所谓的中间临时存储层。其中的 `RingBuffer` 结构我们后面再说。

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

它提供了两个初始化 `UnboundedChan` 的方法，从代码中我们可以明显的看出,`NewUnboundedChanSize` 可以给每个属性自定义自己的容量大小。仅此而已。

`chanx` 中 关于 `in` 和 `out` 都是带缓冲的通道，而上篇文章中的 `in` 和 `out` 都是无缓冲的通道。
这和他们对数据的流转处理有很大关系。


我们接下去看 `process(in,out,ch)` 最核心的方法。
![image](https://image.syst.top/image/unlimited-2/unlimit-1.jpg)

这时候，我们再放上一篇核心代码。

![image](https://image.syst.top/image/unlimited/unlimit-7.jpg)

可以很明显他们看出它俩的区别。

上篇从 `in` 通道读数据会先 `append` 到 `buffer`，然后从 `buffer` 中取数据写入 `out` 通道。
而 `chanx` 从 `in` 通道取出数据先尝试写入 `out`(没有中间商赚差价?)，只有在 `out` 已经满的情况下，才塞入到 `buffer`。

`chanx` 还有一段小细节代码。
![image](https://image.syst.top/image/unlimited-2/unlimit-2.jpg)

能走到这里，一定是因为 `out` 通道满了。我们把值追加到 `buffer` 的同时，需要尝试把 `buffer` 中的数据写入 `out` 。
此时 `in` 通道也许还在持续的写入数据， 为了避免 `in` 通道塞满，阻塞业务写入，我们同时需要尝试从 `in` 通道中读数据追加到 `buffer`。

## buffer

上篇文章我提到了关于 `buffer` 优化的点。

`chanx` 是如何优化的?

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
-  `buf` 具体存储数据的结构。
- `initialSize` 初始化化 `buf` 的长度
- `size` 当前 `buf` 的长度
- `r` 当前读数据位置
- `w` 当前写入数据位置

`buffer` 本质上就是一个环形的队列，目的是达到资源的复用。
并且当 `buffer` 满时，提供自动扩容的功能。

我们来看具体把数据写入 `buffer` 的源码。
![image](https://image.syst.top/image/unlimited-2/unlimit-3.png)

接着看扩容。
![image](https://image.syst.top/image/unlimited-2/unlimit-4.png)

这段代码唯一难理解的就是数据迁移了。这里的数据迁移目的是为了保证先入先出的原则。

可能加了注释有些人也无法理解，那么就再加一个草率图。

假设我们 `buffer` 的长度是 8。 当前读和写的 `index` 都是5。说明 `buffer` 满了，触发自动扩容规则，进行数据迁移。

那么迁移的过程就是下图这样的。

![image](https://image.syst.top/image/unlimited-2/unlimit-5.png)

还有，当 `buffer` 为空并且当前的 `size` 比初始化 `size` 还大，那么可以考虑重置 `buffer` 了。
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
剩下的代码,就没什么好说的了。

### 总结
继上篇文章后，这篇文章我们主要讲解了 `chanx` 是如何实现无限缓冲的 `channel`。
其中最重要的一个点在于 `chanx` 中 `buffer` 实现采用的是 `ringbuffer`，达到资源复用的同时还能自动扩容。



