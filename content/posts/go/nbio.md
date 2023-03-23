---
title: "nbio原理解析"
date: 2023-02-23T23:23:51+08:00 
toc: true 
tags :

- go
- net
- bio

---

之前更新的一系列，好久没更新了，差点烂尾，我不允许这样的事情在我身上发生(虽然已经烂尾好几次了😭)。

在上一篇文章中，我们探讨了基于 epoll 的 Go Netpoll 框架的早期实现——evio。我们还指出了它存在的一些问题。在本篇文章中，我们将继续深入分析另一个高性能的网络编程框架：nbio。

nbio项目里也包含了在nbio之上构建的nbhttp，这个不在我们讨论范围。

nbio同样采用了经典的Reactor模式，事实上，Go语言中的许多异步网络框架都是基于这种模式设计的。

老规矩，先运行nbio程序代码，

Server:

```go
package main

import (
	"fmt"

	"github.com/lesismal/nbio"
)

func main() {
	g := nbio.NewGopher(nbio.Config{
		Network:            "tcp",
		Addrs:              []string{":8888"},
		MaxWriteBufferSize: 6 * 1024 * 1024,
	})

	g.OnData(func(c *nbio.Conn, data []byte) {
		c.Write(append([]byte{}, data...))
	})

	err := g.Start()
	if err != nil {
		fmt.Printf("nbio.Start failed: %v\n", err)
		return
	}
	defer g.Stop()

	g.Wait()
}
```

使用nbio.NewGopher()函数创建一个新的Engine实例。传入nbio.Config结构体来配置 Engine 实例，包括：

- Network: 使用的网络类型，这里是tcp。
- Addrs: 服务器要监听的地址和端口，这里是 ":8888"（即监听本机的8888端口）。
- MaxWriteBufferSize: 写缓冲区的最大大小，这里设置为 6MB。

其他配置可以自行查看。然后使用g.OnData()方法为 Engine实例注册一个数据接收回调函数。这个回调函数在收到数据时被调用。回调函数接收两个参数：连接对象c和收到的数据 data。在回调函数内部，我们使用c.Write()方法将收到的数据原样写回给客户端。

Client:

```go
package main

import (
	"bytes"
	"context"
	"fmt"
	"math/rand"
	"time"

	"github.com/lesismal/nbio"
	"github.com/lesismal/nbio/logging"
)

func main() {
	var (
		ret    []byte
		buf    = make([]byte, 1024*1024*4)
		addr   = "localhost:8888"
		ctx, _ = context.WithTimeout(context.Background(), 60*time.Second)
	)

	logging.SetLevel(logging.LevelInfo)

	rand.Read(buf)

	g := nbio.NewGopher(nbio.Config{})

	done := make(chan int)
	g.OnData(func(c *nbio.Conn, data []byte) {
		ret = append(ret, data...)
		if len(ret) == len(buf) {
			if bytes.Equal(buf, ret) {
				close(done)
			}
		}
	})

	err := g.Start()
	if err != nil {
		fmt.Printf("Start failed: %v\n", err)
	}
	defer g.Stop()

	c, err := nbio.Dial("tcp", addr)
	if err != nil {
		fmt.Printf("Dial failed: %v\n", err)
	}
	g.AddConn(c)
	c.Write(buf)

	select {
	case <-ctx.Done():
		logging.Error("timeout")
	case <-done:
		logging.Info("success")
	}
}
```

乍一看有点麻烦。其实是服务端和客户端共用了一套结构。

客户端通过nbio.Dial连接服务端，连接成功封装成nbio.Conn，这里的nbio.Conn是实现了标准库的net.Conn 接口的。最后AddConn这个conn。调用Write往服务端写数据，当服务端接收到数据后，Server端的处理逻辑是把数据原样发送给客户端，当客户端接收到数据一样OnData会被回调，上面OnData判断接受到的数据和发送的数据是不是长度一样，最后客户端主动关闭这个连接。



下面来看几个主要结构。

```go
type Engine struct {
//..
	sync.WaitGroup

//..

	mux sync.Mutex

	wgConn sync.WaitGroup

	network   string
	addrs     []string
	listen    func(network, addr string) (net.Listener, error)


	pollerNum                    int
	readBufferSize               int
	maxWriteBufferSize           int
	maxConnReadTimesPerEventLoop int
	udpReadTimeout               time.Duration
	epollMod                     uint32
/..

	connsStd  map[*Conn]struct{}
	connsUnix []*Conn

	listeners []*poller
	pollers   []*poller

	onOpen            func(c *Conn)
	onClose           func(c *Conn, err error)
	onRead            func(c *Conn)
	onData            func(c *Conn, data []byte)
	onReadBufferAlloc func(c *Conn) []byte
	onReadBufferFree  func(c *Conn, buffer []byte)
	//...
}
```

Engine本质上就是核心管理器。会管理所有的listener poller 和worker poller。

这两种poller有什么区别吗？

区别在职责上。

listener poller只负责accept新的连接，当一个新的客户端conn到来时，会从pollers挑选一个worker poller，然后把conn加入到对应的worker poller，之后worker poller负责处理此conn的读写事件。

所以当我们启动程序的时候，如果只监听一个地址的情况下，那么程序的poll数= 1(listener poller) + pollerNum。

从上面的字段也可以看出，你可以自定义一些配置和回调。比如你可以设置当新连接到来时的回调函数onOpen，也可以设置一个conn数据到来时的回调函数onData等。

```go

type Conn struct {
	mux sync.Mutex

	p *poller

	fd int

  //...

	writeBuffer []byte

	typ      ConnType
	closed   bool
	isWAdded bool
	closeErr error

	lAddr net.Addr
	rAddr net.Addr

	ReadBuffer []byte
//...

	DataHandler func(c *Conn, data []byte)
}
```

 Conn结构体，用于表示一个网络连接。一个conn只属于一个poller。对应的writeBuffer:当数据一次没写完时，剩下的先存在writeBuffer，等待下次可写事件到来继续写入。

```go
type poller struct {
	g *Engine

	epfd  int
	evtfd int

	index int

	shutdown bool

	listener     net.Listener
	isListener   bool
	unixSockAddr string

	ReadBuffer []byte

	pollType string
}
```

至于poller结构，这里就是一个抽象的概念，用于管理底层的多路复用 I/O（如linux的epoll、darwin上的kqueue 等）

注意这里的pollType，nbio默认epoll采用的是水平触发(LT)，当然用户也可以设置成边缘触发(ET)。

介绍完基本的结构，接下来进入代码的流程。

上面服务端的代码，当你调用Start启动程序，

```go
func (g *Engine) Start() error {
//....
	switch g.network {
    
    //第一部分 初始化listener
	case "unix", "tcp", "tcp4", "tcp6":
		for i := range g.addrs {
			// listener poller
			ln, err := newPoller(g, true, i)
			if err != nil {
				for j := 0; j < i; j++ {
					g.listeners[j].stop()
				}
				return err
			}
			g.addrs[i] = ln.listener.Addr().String()
			g.listeners = append(g.listeners, ln)
		}
	
    //.....

   //第二部分 初始化一定数量poller
	for i := 0; i < g.pollerNum; i++ {
		// rw poller
		p, err := newPoller(g, false, i)
		if err != nil {
			for j := 0; j < len(g.listeners); j++ {
				g.listeners[j].stop()
			}

			for j := 0; j < i; j++ {
				g.pollers[j].stop()
			}
			return err
		}
		g.pollers[i] = p
	}

    //第三部分 启动所有的worker poller
	for i := 0; i < g.pollerNum; i++ {
		g.pollers[i].ReadBuffer = make([]byte, g.readBufferSize)
		g.Add(1)
		go g.pollers[i].start()
	}

    // 第四部分 启动所有的listenr
	for _, l := range g.listeners {
		g.Add(1)
		go l.start()
	}

	//...忽略udp
  // ....
}

```

代码还是易懂的，整体看就四个部分。

**第一部分：初始化 listener**

根据 `g.network` 的值（如 "unix", "tcp", "tcp4", "tcp6"），为每个要监听的地址（`g.addrs`）创建一个新的 `poller`。这里的 `poller`主要用于管理监听套接字上的事件。如果创建 `poller` 时出错，将停止之前创建的所有监听器并返回错误。

**第二部分：初始化一定数量的 poller**

根据pollerNum创建对应个数的worker poller。这些 `poller` 用于处理已连接套接字上的读/写事件。如果在创建过程中遇到错误，将停止所有监听器和之前创建的工作 `poller`，然后返回错误。

**第三部分：启动所有的 worker poller**

为每个工作 `poller` 分配一个读缓冲区（由 `g.readBufferSize` 决定大小），并发地启动这些 `poller`。

**第四部分：启动所有的 listener**

启动所有之前创建的监听器。开始监听对应地址的连接请求。

至于poller的启动，

```go
func (p *poller) start() {
	defer p.g.Done()
  //...
	if p.isListener {
		p.acceptorLoop()
	} else {
		defer func() {
			syscall.Close(p.epfd)
			syscall.Close(p.evtfd)
		}()
		p.readWriteLoop()
	}
}
```

分为两种，如果是一个listener poller，

```go
func (p *poller) acceptorLoop() {
   //如果想要当前g不被调度到其他的操作线程。
	if p.g.lockListener {
		runtime.LockOSThread()
		defer runtime.UnlockOSThread()
	}

	p.shutdown = false
	for !p.shutdown {
		conn, err := p.listener.Accept()
		if err == nil {
			var c *Conn
			c, err = NBConn(conn)
			if err != nil {
				conn.Close()
				continue
			}
      //
			p.g.pollers[c.Hash()%len(p.g.pollers)].addConn(c)
		} else {
			var ne net.Error
			if ok := errors.As(err, &ne); ok && ne.Timeout() {
				logging.Error("NBIO[%v][%v_%v] Accept failed: temporary error, retrying...", p.g.Name, p.pollType, p.index)
				time.Sleep(time.Second / 20)
			} else {
				if !p.shutdown {
					logging.Error("NBIO[%v][%v_%v] Accept failed: %v, exit...", p.g.Name, p.pollType, p.index, err)
				}
				break
			}
		}
	}
}
```

listener poller 就是等待新连接，然后通过NBConn封装成nbio conn结构，最后通过取模操作获取其中一个woker poller。把连接加入到对应的poller中。

```go
func (p *poller) addConn(c *Conn) {
   c.p = p
   if c.typ != ConnTypeUDPServer {
      p.g.onOpen(c)
   }
   fd := c.fd
   p.g.connsUnix[fd] = c
   err := p.addRead(fd)
   if err != nil {
      p.g.connsUnix[fd] = nil
      c.closeWithError(err)
      logging.Error("[%v] add read event failed: %v", c.fd, err)
   }
}
}
```

这里一个有趣的设计，在管理conns上，结构是slice，作者直接使用的conn的fd来作为下标。

这样还是有好处的,

- 连接超多的情况下，GC的时候负担会比map小。
- 能防止串号问题。

最后通过调用addRead把对应的conn fd加入到epoll。

```go
func (p *poller) addRead(fd int) error {
	switch p.g.epollMod {
	case EPOLLET:
		return syscall.EpollCtl(p.epfd, syscall.EPOLL_CTL_ADD, fd, &syscall.EpollEvent{Fd: int32(fd), Events: syscall.EPOLLERR | syscall.EPOLLHUP | syscall.EPOLLRDHUP | syscall.EPOLLPRI | syscall.EPOLLIN | syscall.EPOLLET})
	default:
		return syscall.EpollCtl(p.epfd, syscall.EPOLL_CTL_ADD, fd, &syscall.EpollEvent{Fd: int32(fd), Events: syscall.EPOLLERR | syscall.EPOLLHUP | syscall.EPOLLRDHUP | syscall.EPOLLPRI | syscall.EPOLLIN})
	}
}
```

这里没有注册写事件是合理的，因为在新连接上还没有收到任何数据，所以暂时没有需要发送的数据。这种做法可以避免一些不必要的系统调用，从而提高程序的性能。

如果是worker poller的启动，它的工作就是等待加入的那些conns的事件到来，进行对应的处理。

```go
func (p *poller) readWriteLoop() {
  
  //....
   msec := -1
   
  events := make([]syscall.EpollEvent, 1024)
  // ......
   for !p.shutdown {
      n, err := syscall.EpollWait(p.epfd, events, msec)
      if err != nil && !errors.Is(err, syscall.EINTR) {
         return
      }

      if n <= 0 {
         msec = -1
         // runtime.Gosched()
         continue
      }
      msec = 20

     // 遍历事件
      for _, ev := range events[:n] {
         fd := int(ev.Fd)
         switch fd {
         case p.evtfd:
         default:
            c := p.getConn(fd)
            if c != nil {
               if ev.Events&epollEventsError != 0 {
                  c.closeWithError(io.EOF)
                  continue
               }

               // 说明可写 刷数据
               if ev.Events&epollEventsWrite != 0 {
                  c.flush()
               }

               //读事件
               if ev.Events&epollEventsRead != 0 {
                  if p.g.onRead == nil {
                     for i := 0; i < p.g.maxConnReadTimesPerEventLoop; i++ {
                        buffer := p.g.borrow(c)
                        rc, n, err := c.ReadAndGetConn(buffer)
                        if n > 0 {
                           p.g.onData(rc, buffer[:n])
                        }
                        p.g.payback(c, buffer)
                        // ...
                        if n < len(buffer) {
                           break
                        }
                     }
                  } else {
                     p.g.onRead(c)
                  }
               }
            } else {
               syscall.Close(fd)
               // p.deleteEvent(fd)
            }
         }
      }
   }
}
```

这段代码也很好理解。等待事件到来，遍历对应的事件列表，判断事件类型，相对应的处理。

```go
func EpollWait(epfd int, events []EpollEvent, msec int) (n int, err error)
```

EpollWait中只有msec是可以用户动态修改的，通常情况下，我们主动调用EpollWait都会设置msec=-1，msce=-1会使得函数一直等待，直到至少有一个事件发生，否则的话一直阻塞。这种方法在事件发生较少的情况下非常有用，因为它可以最大限度地减少 CPU占用率。

如果希望尽可能快速响应事件，可以将msec设置为 0。这将使 EpollWait立即返回，不等待任何事件。这种情况下，你的程序可能会更频繁地调用EpollWait，但能够在事件发生后立即处理它们。当然，这就会导致CPU占用率较高。

如果你的程序可以承受一定的延迟，并希望减少 CPU占用率，可以将msec设置为一个正数。这将使得EpollWait在指定的时间内等待事件。如果在这段时间内没有事件发生，函数将返回，你可以选择在稍后再次调用EpollWait。这种方法可以降低 CPU占用率，但可能会导致较长的响应时间。

nbio对应这个值的调整策略是:当事件数量大于0时，msec=20(这个20应该是作者测试后综合考量?)。

字节跳动的netpoll代码是这样的，如果事件数量大于0，会设置msec为0。如果事件小于等于0，设置msec=-1，然后调用Gosched()使得当前Goroutine主动让出P。

```go
var msec = -1
for {
   n, err = syscall.EpollWait(epfd, events, msec)
   if n <= 0 {
      msec = -1
      runtime.Gosched()
      continue
   }
   msec = 0
   ...
}

```

然而，nbio 中主动切换的代码已经被注释掉了。根据作者在 issue 中的解释，最初他参考了字节的方法加入了主动切换。

但在对 nbio 进行性能测试时，发现加入与不加入主动切换对性能没有明显区别，因此最终决定将其移除。



事件的处理部分，

如果是可读事件，你可以通过内置或者自定义的内存分配器来得到相应的buffer，然后调用ReadAndGetConn读取数据，而不需要每次都申请一遍buffer。

如果是可写事件的话，会调用flush把buffer里未发送的数据发送出去。

```go
func (c *Conn) flush() error {
  //.....
	old := c.writeBuffer
	n, err := c.doWrite(old)
	if err != nil && !errors.Is(err, syscall.EINTR) && !errors.Is(err, syscall.EAGAIN) {
    //.....
	}

	if n < 0 {
		n = 0
	}
	left := len(old) - n
  // 说明没写完，把剩下的存进writeBuffer下次写
	if left > 0 {
		if n > 0 {
			c.writeBuffer = mempool.Malloc(left)
			copy(c.writeBuffer, old[n:])
			mempool.Free(old)
		}
		// c.modWrite()
	} else {
		mempool.Free(old)
		c.writeBuffer = nil
		if c.wTimer != nil {
			c.wTimer.Stop()
			c.wTimer = nil
		}
    // 说明写完了，先把conn重置为只有读事件
		c.resetRead()
    //...
	}

	c.mux.Unlock()
	return nil
}
```

逻辑也很简单，写多少是多少，写不进去把剩下的重新放入writeBuffer，下轮epollWait触发再写。

如果写完了，那就没数据可写了，重置这个conn的事件为读事件。

主逻辑就差不多是这样了。

等等，我们一开始说一个新连接进来的时候，我们对一个连接只注册了读事件，没注册写事件，写事件是什么时候注册的？

当然是你调用conn.Write的时候，

```go
g := nbio.NewGopher(nbio.Config{
		Network:            "tcp",
		Addrs:              []string{":8888"},
		MaxWriteBufferSize: 6 * 1024 * 1024,
	})

	g.OnData(func(c *nbio.Conn, data []byte) {
		c.Write(append([]byte{}, data...))
	})

```

当conn的数据到来时，底层读完数据，会回调OnData函数，此时你可以调用Write向对端发送数据，

```go
func (c *Conn) Write(b []byte) (int, error) {
  //....
	n, err := c.write(b)
	if err != nil && !errors.Is(err, syscall.EINTR) && !errors.Is(err, syscall.EAGAIN) {
		//.....
		return n, err
	}

	if len(c.writeBuffer) == 0 {
		if c.wTimer != nil {
			c.wTimer.Stop()
			c.wTimer = nil
		}
	} else {
    // 说明还有数据没写完，加入写事件
		c.modWrite()
	}
  //.....
	return n, err
}

func (c *Conn) write(b []byte) (int, error) {
  //...
	if len(c.writeBuffer) == 0 {
		n, err := c.doWrite(b)
		if err != nil && !errors.Is(err, syscall.EINTR) && !errors.Is(err, syscall.EAGAIN) {
			return n, err
		}
    //.....
    
		left := len(b) - n
    //还没写完，剩下的放入writeBuffer
		if left > 0 && c.typ == ConnTypeTCP {
			c.writeBuffer = mempool.Malloc(left)
			copy(c.writeBuffer, b[n:])
			c.modWrite()
		}
		return len(b), nil
	}
  // 如果本身writeBuffer还有数据没写入，那新的数据也append进来
	c.writeBuffer = mempool.Append(c.writeBuffer, b...)

	return len(b), nil
}
```

当数据没写完，就把剩余数据放入到writeBuffer，这样就会触发执行modWrite，就会把conn的写事件注册到epoll中。



总结

相较于evio，nbio不会出现惊群效应。

evio是通过不断无效的唤醒epoll，来达到逻辑的正确性。而nbio是尽可能的减少系统调用，减少无谓的开销。

易用性上，nbio实现了标准库net.Conn，同时很多设置可配置化，用户可以自由定制，自由度较高。

读写上会使用预先分配的buffer，提高应用性能。

总之，nbio是一款不错的高性能非阻塞的网络框架。





