---
title: "evio原理解析～有彩蛋"
date: 2023-01-06T19:23:51+08:00 
toc: true 
tags :

- go
- net

---

之前分析过go自带的netpoll，以及自建的网络框架gnet。

当然这类框架还有:evio、gev、nbio、cloudwego/netpoll(字节的)。

为什么会出现这么多自建框架?

我觉得逃不过三点，

- 自带的netpoll满足不了一些特殊场景。

- 其他实现设计存在局限性，存在优化空间。
- 程序员都喜欢自己造轮子。

另外，这类框架都是基于syscall epoll实现的事件驱动框架。主要区别我觉得在于，

- 对连接conn的管理
- 对读写数据管理

带着这些问题，我打算把这些框架都看一遍。学习里面优秀的设计以及对比他们的不同点，可以的话，做个整体的性能测试。

这几个框架中，evio是最早的开源实现，开源于2017年。

有意思的是，看到几篇文章说evio存在当loopWrite在内核缓冲区满，无法一次写入时，会出现写入数据丢失的bug。

![](https://cdn.syst.top/evio3.png)

仔细阅读了代码，evio并不存在这个bug。也不存在是作者后来修复了这个bug，而是evio本身不存在这个bug。

下面会说明。



#### 原理解析

根据代码画了个简易架构图说明evio架构。

![](https://cdn.syst.top/evio1.png)

简单解释一下，evio启动的时候可以指定loops个数，即多少个epoll实例。同时可以启动多个监听地址，比如图中监听了两个端口。

程序会把每个Listener fd加入到每个epoll并注册这些fd的读事件。每个epoll会开启一个goroutine等待事件到来。

当客户端发起对应端口连接，程序会根据策略选择一个epoll，并把conn fd 也加入到此epoll并注册读写事件。

当一个conn fd读事件ready，那么对应的epoll会被唤醒，然后执行相应的操作。



以上就是整理的流程，接下来我们来深入一些细节。

在此之前，根据上面所描述的，需要先提几个问题，

- 当一个新的客户端连接到来时，会发生什么？
- 读写数据是如何流动的？
- 同一个epoll里多个fd读写事件ready，程序是如何处理的？

看完下面，再回来回答这三个问题。



##### 代码细节

运行一个简单demo，

```go
package main

import (
	"fmt"
	"github.com/tidwall/evio"
	"log"
)

func main() {
	var events evio.Events
	events.NumLoops = 3
	events.Serving = func(srv evio.Server) (action evio.Action) {
		log.Printf("echo server started")
		return
	}
	events.Data = func(c evio.Conn, in []byte) (out []byte, action evio.Action) {
		fmt.Println("receiver data:", string(in))
		out = in
		return
	}
	log.Fatal(evio.Serve(events, "tcp://:8089", "tcp://:8088"))
}
```

我们指定NumLoops的数量是3，然后传入了两个地址。

上面还有两个闭包函数，当服务启动的时候会回调events.Serving函数，然后返回一个Action。

```go
const (
	// None indicates that no action should occur following an event.
	None Action = iota
	// Detach detaches a connection. Not available for UDP connections.
	Detach
	// Close closes the connection.
	Close
	// Shutdown shutdowns the server.
	Shutdown
)
```

比如当你返回一个Shutdown的action，那么程序就直接退出了。

当有客户端数据到来时，回调events.Data函数，返回out和action，out表示要发送的数据。

最终调用 evio.Serve函数，传入两个地址，启动服务。

```go
func Serve(events Events, addr ...string) error {
	var lns []*listener
	defer func() {
		for _, ln := range lns {
			ln.close()
		}
	}()
	for _, addr := range addr {
		var ln listener
		ln.network, ln.addr, ln.opts, _= parseAddr(addr)
		
		if ln.network == "unix" {
			os.RemoveAll(ln.addr)
		}
		var err error
		if ln.network == "udp" {
			if ln.opts.reusePort {
				ln.pconn, err = reuseportListenPacket(ln.network, ln.addr)
			} else {
				ln.pconn, err = net.ListenPacket(ln.network, ln.addr)
			}
		} else {
			if ln.opts.reusePort {
				ln.ln, err = reuseportListen(ln.network, ln.addr)
			} else {
				ln.ln, err = net.Listen(ln.network, ln.addr)
			}
		}
		if err != nil {
			return err
		}
		if ln.pconn != nil {
			ln.lnaddr = ln.pconn.LocalAddr()
		} else {
			ln.lnaddr = ln.ln.Addr()
		}
		lns = append(lns, &ln)
	}
	return serve(events, lns)
}
```

我删除了一些无关代码。

Serve 函数里面遍历传入的地址识别协议，执行对应listen操作。udp返回一个PacketConn，而tcp返回一个Listener。最终用自定义的listener统一表示。最终调用serve。

````go
func serve(events Events, listeners []*listener) error {
	// figure out the correct number of loops/goroutines to use.
	numLoops := events.NumLoops
	if numLoops <= 0 {
		if numLoops == 0 {
			numLoops = 1
		} else {
			numLoops = runtime.NumCPU()
		}
	}

	s := &server{}
	s.events = events
	s.lns = listeners
	s.cond = sync.NewCond(&sync.Mutex{})
	s.balance = events.LoadBalance
	s.tch = make(chan time.Duration)

	//println("-- server starting")
	if s.events.Serving != nil {
		var svr Server
		svr.NumLoops = numLoops
		svr.Addrs = make([]net.Addr, len(listeners))
		for i, ln := range listeners {
			svr.Addrs[i] = ln.lnaddr
		}
		action := s.events.Serving(svr)
		switch action {
		case None:
		case Shutdown:
			return nil
		}
	}

	defer func() {
		// wait on a signal for shutdown
		s.waitForShutdown()

		// notify all loops to close by closing all listeners
		for _, l := range s.loops {
			l.poll.Trigger(errClosing)
		}

		// wait on all loops to complete reading events
		s.wg.Wait()

		// close loops and all outstanding connections
		for _, l := range s.loops {
			for _, c := range l.fdconns {
				loopCloseConn(s, l, c, nil)
			}
			l.poll.Close()
		}
		//println("-- server stopped")
	}()

	// create loops locally and bind the listeners.
	for i := 0; i < numLoops; i++ {
		l := &loop{
			idx:     i,
			poll:    internal.OpenPoll(),
			packet:  make([]byte, 0xFFFF),
			fdconns: make(map[int]*conn),
		}
		for _, ln := range listeners {
			l.poll.AddRead(ln.fd)
		}
		s.loops = append(s.loops, l)
	}
	// start loops in background
	s.wg.Add(len(s.loops))
	for _, l := range s.loops {
		go loopRun(s, l)
	}
	return nil
}
````

这个函数逻辑:

1.先初始化一个自定义server结构，确定负载均衡算法。

2.回调自定义的serving闭包函数。

3.根据numloops值，创建对应数量的epoll封装在自定义结构loop中。并把每一个listener对应的fd加入到每一个epoll同时注册fd的读事件，

```go
for _, ln := range listeners {
			l.poll.AddRead(ln.fd)
		}

func (p *Poll) AddRead(fd int) {
	if err := syscall.EpollCtl(p.fd, syscall.EPOLL_CTL_ADD, fd,
		&syscall.EpollEvent{Fd: int32(fd),
			Events: syscall.EPOLLIN, //读事件
		},
	); err != nil {
		panic(err)
	}
}
```

4.遍历loops,每个loop都用一个g执行loopRun函数。



至于loopRun函数，

```go
func loopRun(s *server, l *loop) {
	defer func() {
		//fmt.Println("-- loop stopped --", l.idx)
		s.signalShutdown()
		s.wg.Done()
	}()

	if l.idx == 0 && s.events.Tick != nil {
		go loopTicker(s, l)
	}

	//fmt.Println("-- loop started --", l.idx)
	l.poll.Wait(func(fd int, note interface{}) error {
		if fd == 0 {
			return loopNote(s, l, note)
		}
		c := l.fdconns[fd]
		switch {
		case c == nil:
			return loopAccept(s, l, fd)
		case !c.opened:
			return loopOpened(s, l, c)
		case len(c.out) > 0:
			return loopWrite(s, l, c)
		case c.action != None:
			return loopAction(s, l, c)
		default:
			return loopRead(s, l, c)
		}
	})
}
```

每个loop都会调用epoll.Wait函数阻塞等待事件到来，参数时一个闭包函数，每一个到达的事件都会回调此闭包执行相应操作。

```go
func (p *Poll) Wait(iter func(fd int, note interface{}) error) error {
	events := make([]syscall.EpollEvent, 64)
	for {
    // 等待事件
		n, err := syscall.EpollWait(p.fd, events, 100)
		if err != nil && err != syscall.EINTR {
			return err
		}
		if err := p.notes.ForEach(func(note interface{}) error {
			return iter(0, note)
		}); err != nil {
			return err
		}
		for i := 0; i < n; i++ {
      // 正常的客户端fd
			if fd := int(events[i].Fd); fd != p.wfd {
				if err := iter(fd, nil); err != nil {
					return err
				}
			} else if fd == p.wfd {
				var data [8]byte
				syscall.Read(p.wfd, data[:])
			}
		}
	}
}
```

回到上一步，

```go
l.poll.Wait(func(fd int, note interface{}) error {
		if fd == 0 {
			return loopNote(s, l, note)
		}
		c := l.fdconns[fd]
		switch {
		case c == nil:
			return loopAccept(s, l, fd)
		case !c.opened:
			return loopOpened(s, l, c)
		case len(c.out) > 0:
			return loopWrite(s, l, c)
		case c.action != None:
			return loopAction(s, l, c)
		default:
			return loopRead(s, l, c)
		}
	})
```

注意这里是每个loop都会调用自己的epoll.Wait。当对应的Listener来了一个新客户端连接，所有的epoll都会被“惊醒”，这就是惊群效应。

> 惊群效应（thundering herd）是指多进程（多线程）在同时阻塞等待同一个事件的时候（休眠状态），如果等待的这个事件发生，那么它就会唤醒等待的所有进程（或者线程）

然后所有的loop都会执行loopAccept函数，

```go
func loopAccept(s *server, l *loop, fd int) error {
   for i, ln := range s.lns {
      // 确实是哪个listener fd 事件
      if ln.fd == fd {
         if len(s.loops) > 1 {
            switch s.balance {
            case LeastConnections:
               n := atomic.LoadInt32(&l.count)
               for _, lp := range s.loops {
                  // 对比其他的loop，有比自己连接数少的，自己就不连接了
                  if lp.idx != l.idx {
                     if atomic.LoadInt32(&lp.count) < n {
                        return nil // do not accept
                     }
                  }
               }
            case RoundRobin:
               idx := int(atomic.LoadUintptr(&s.accepted)) % len(s.loops)
               //还没轮到我，也不能连接
               if idx != l.idx {
                  return nil // do not accept
               }
               atomic.AddUintptr(&s.accepted, 1)
            }
         }

         // udp
         if ln.pconn != nil {
            return loopUDPRead(s, l, i, fd)
         }

         nfd, sa, err := syscall.Accept(fd)
         if err != nil {
            if err == syscall.EAGAIN {
               return nil
            }
            return err
         }
         //设置 conn fd 为非阻塞
         if err := syscall.SetNonblock(nfd, true); err != nil {
            return err
         }
         c := &conn{fd: nfd, sa: sa, lnidx: i, loop: l}
         c.out = nil
         l.fdconns[c.fd] = c
         //把conn fd 注册到此epoll，且监听读写事件
         l.poll.AddReadWrite(c.fd)
         atomic.AddInt32(&l.count, 1)
         break
      }
   }
   return nil
}
```

loopAccept做了几件事：

1.确定就绪的fd是哪个listener

2.如果loops大于1，通过策略选择其中一个loop,包装conn fd且设置conn fd为非阻塞(思考下如果让conn fd保持阻塞状态，会影响到什么？)

3.最后把这个conn fd加入到当前epoll并且注册读写事件



因此当一个新的客户端连接到来时，会发生什么？

会产生惊群效应。

那如果是已存在的conn fd的可读事件，会发生惊群效应吗？

不会，因为一个conn fd只会加入到其中一个epoll中。



因为evio创建epoll的时候默认是水平触发LT(level-triggered)，当加入的conn fd包含写事件时，如果此时内核写缓冲区空间未满，那么epoll会再次被唤醒。此时通过f.fdconns[fd]会找到对应的conn，由于连接初始化并未设置opened的直，因此会进入loopOpened函数。

```go
func loopOpened(s *server, l *loop, c *conn) error {
	c.opened = true
	c.addrIndex = c.lnidx
	c.localAddr = s.lns[c.lnidx].lnaddr
	c.remoteAddr = internal.SockaddrToAddr(c.sa)
	if s.events.Opened != nil {
		out, opts, action := s.events.Opened(c)
		if len(out) > 0 {
			c.out = append([]byte{}, out...)
		}
		c.action = action
		c.reuse = opts.ReuseInputBuffer
		if opts.TCPKeepAlive > 0 {
			if _, ok := s.lns[c.lnidx].ln.(*net.TCPListener); ok {
				internal.SetKeepAlive(c.fd, int(opts.TCPKeepAlive/time.Second))
			}
		}
	}
	// 没有可写的数据，并且也未设置连接的action，重新修改此conn只有read事件
	if len(c.out) == 0 && c.action == None {
		l.poll.ModRead(c.fd)
	}
	return nil
}
```

无非是一些简单赋值操作，如果设置了Opened闭包函数，那么回调它。

最后如果out没有可发送的事件，那么就重新把此conn fd修改成读事件。

想象一下，如果在没有可写数据的情况下，加入epoll的conn fd注册含有写事件，那么只要内核写缓存区未满，此epoll会不断被唤醒，我称它是空转。

如果上面你设置了Opened闭包函数且最终action设置了值，那么就还是保持此conn fd的读写事件。

这时候再次唤醒的epoll会执行loopAction，

```go
func loopAction(s *server, l *loop, c *conn) error {
   switch c.action {
   default:
      c.action = None
   case Close:
      return loopCloseConn(s, l, c, nil)
   case Shutdown:
      return errClosing
   case Detach:
      return loopDetachConn(s, l, c, nil)
   }
   if len(c.out) == 0 && c.action == None {
      l.poll.ModRead(c.fd)
   }
   return nil
}
```

上面代码很简单。

当client发送数据，epoll被唤醒，执行loopRead。

```
func loopRead(s *server, l *loop, c *conn) error {
   var in []byte
   // 读数据
   n, err := syscall.Read(c.fd, l.packet)
   if n == 0 || err != nil {
      if err == syscall.EAGAIN {
         return nil
      }
      return loopCloseConn(s, l, c, err)
   }
   in = l.packet[:n]
   if !c.reuse {
      in = append([]byte{}, in...)
   }
   // 回调data闭包函数
   if s.events.Data != nil {
      out, action := s.events.Data(c, in)
      c.action = action
      // 表示要发送的数据
      if len(out) > 0 {
         c.out = append(c.out[:0], out...)
      }
   }
   // 说明有数据要发送，重新注册读写事件
   if len(c.out) != 0 || c.action != None {
      l.poll.ModReadWrite(c.fd)
   }
   return nil
}
```

先读取数据，如果有数据要发送，又会修改conn fd为读写事件。当epoll再次被唤醒，且c.out不为空时，执行loopWrite。

```go
func loopWrite(s *server, l *loop, c *conn) error {
	if s.events.PreWrite != nil {
		s.events.PreWrite()
	}
	n, err := syscall.Write(c.fd, c.out)
	if err != nil {
		if err == syscall.EAGAIN {
			return nil
		}
		return loopCloseConn(s, l, c, err)
	}
	// 说明一次就写完数据了
	if n == len(c.out) {
		// release the connection output page if it goes over page size,
		// otherwise keep reusing existing page.
		// 不让out无限增长空间
		if cap(c.out) > 4096 {
			c.out = nil
		} else {
			// 重置nil，复用
			c.out = c.out[:0]
		}
	} else {
		//没写完，留着后面没写的部分
		c.out = c.out[n:]
	}
	// 全写完了，重新调整为读事件
	if len(c.out) == 0 && c.action == None {
		l.poll.ModRead(c.fd)
	}
	return nil
}
```

如果一次写完数据，那么说明暂时没有可写的数据了，重新修改conn fd为读事件。

如果一次没写完，那么保留未写完的数据，下次epoll唤醒的时候继续写。

上面提到，一些文章提到，evio存在 loopWrite在内核缓冲区满，无法一次写入时，会出现写入数据丢失的bug。

给的理由是，当内核写缓冲区满了，可数据并未写完。此时另一个conn读事件ready，会执行loopRead。

loopRead有这么一段代码，

```go
if s.events.Data != nil {
		out, action := s.events.Data(c, in)
		c.action = action
		// 表示要发送的数据
		if len(out) > 0 {
      // 之前这里是:c.out = append([]byte{}, out...)，效果是一样的
			c.out = append(c.out[:0], out...)
		}
	}
```

那么此时之前未发送完的数据就会被覆盖，导致数据丢失。

但是我仔细看了代码，并不存在这个bug。

因为如果第一次没写完，假设此时同一个epoll下的另一个conn读事件ready，由于out还有未写完的数据，只会执行loopWrite分支， 并不会走到默认分支loopRead，也就不存在写数据被覆盖导致丢失的问题了。

有趣的是，这样的情况会导致空转。因为执行loopWrite逻辑，由于内核写缓冲区已满，导致写不进去数据，会出现syscall.EAGAIN直接返回。又因此时还有可读的数据没读，会不断唤醒epoll。

调用epoll_wait会陷入内核态，所以会导致不断的在用户态和内核态切换。直到写完数据，才能读其他conn数据。

到这里，核心的代码已经分析完了。



#### Evio存在的问题

**惊群效应**。

**串行化**

从上面的分析可以看出，如果一个epoll有两个fd可读事件ready，那么第二个fd必须等第一个执行完毕，才开始执行。

换句话说，如果这个闭包函数里有外部依赖调用，第二个就得一直等。

不能在Data函数里用go func吗？

还真不能，要是这样的话又会涉及到数据并发问题，数据会发生错乱。

同一个epoll下的conn是共享数据结构的，如果使用异步，必然又涉及到锁的问题。

**数据copy问题**

evio采用的是同步处理buffer数据，直接通过syscall读写操作存在copy开销，这是cpu直接参与的。看字节的netpoll使用的zero-copy的技术，后面再看源码。

**频繁唤醒epoll**

evio会通过不断修改conn fd的事件来唤醒epoll，达到逻辑上的正确性。频繁唤醒的方式并不是很妥，这种方式是存在开销的。



这篇文章到这里就结束了，分析完go自建netpoll "鼻祖" evio，以及它存在局限性，就可以继续学习后续其他框架的设计了。








