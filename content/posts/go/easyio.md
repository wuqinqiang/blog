---
title: "easyio:最小化的netpoll实现"
date: 2023-06-03T17:06:51+08:00 
toc: true 
tags :

- go
- netpoll
-
---



目前Go圈有很多款异步的网络框架:

- https://github.com/tidwall/evio
- https://github.com/lesismal/nbio
- https://github.com/panjf2000/gnet
- https://github.com/cloudwego/netpoll
- .......

排名不分先后。

这里面最早的实现是evio。evio也存在一些问题，之前也写过[evio](https://www.syst.top/posts/go/evio/)文章介绍过。 其他比如nbio和gnet也写过一些源码分析。

为什么会出现这些框架？之前也提到过，由于标准库netpoll的一些特性:

- 一个conn一个goroutine导致利用率低
- 用户无法感知conn状态
- .....

这些框架在应用层上做了很多优化，比如:Worker Pool,Buffer,Ring Buffer,NoCopy......。

都分析了好几篇的代码了，那么咋么说也得自己动手搞一个来达成学习目的。

没错，这就是easyio的由来。

它是一个最小化的IO框架，只实现最核心的部分，加起来不超过500行代码。

也没有用户端上层应用的优化，且目前只实现了linux的epoll，以及只能运行tcp协议。

大概结构如下，具体可以看代码～～，

![截屏2023-06-03 23.28.59](https://cdn.syst.top/%E6%88%AA%E5%B1%8F2023-06-11%2010.19.57.png)



简单的demo，

服务端:

```go
package main

import (
	"context"
	"fmt"
	"os"
	"os/signal"
	"syscall"

	"github.com/wuqinqiang/easyio"
)

var _ easyio.EventHandler = (*Handler)(nil)

type Handler struct{}

type EasyioKey struct{}

type Message struct{ Msg string }

var CtxKey EasyioKey

func (h Handler) OnOpen(c easyio.Conn) context.Context {
	return context.WithValue(context.Background(), CtxKey, Message{Msg: "helloword"})
}

func (h Handler) OnRead(ctx context.Context, c easyio.Conn) {
	_, ok := ctx.Value(CtxKey).(Message)
	if !ok {
		return
	}
	var b = make([]byte, 100)
	_, err := c.Read(b)
	if err != nil {
		fmt.Println("err:", err)
	}
	fmt.Println("[Handler] read data:", string(b))

	if _, err = c.Write(b); err != nil {
		panic(err)
	}
}

func (h Handler) OnClose(_ context.Context, c easyio.Conn) {
	fmt.Println("[Handler] closed", c.Fd())
}

func main() {
	e := easyio.New("tcp", ":8090",easyio.WithNumPoller(4), easyio.WithEventHandler(Handler{}))

	if err := e.Start(); err != nil {
		panic(err)
	}

	defer e.Stop()

	c := make(chan os.Signal, 1)
	signal.Notify(c, syscall.SIGTERM, syscall.SIGQUIT, syscall.SIGINT)
	<-c
}

```

上面的代码，初始化一个easyio，启动一个tcp服务，监听端口8090，options里面设置epoll的数量，以及设置事件处理器。

当一个新连接到来时会回调 OnOpen函数，此时你可以设置自定义的ctx，那么当对应连接读事件到来OnRead回调，你可以拿到之前设置的ctx，调用conn.Read读取数据，且通过Write向对端写数据。

这里需要注意的是，一个连接如果数据没读完，当OnRead执行结束，下一轮会继续触发回调代码，因为底层epoll采用的是LT触发方式。



简单的客户端

```go
package main

import (
	"fmt"
	"net"
	"os"
	"os/signal"
	"syscall"
)

func main() {
	conn, err := net.Dial("tcp", ":8090")
	if err != nil {
		panic(err)
	}
	n, err := conn.Write([]byte("hello world"))
	if err != nil {
		panic(err)
	}

	go func() {
		b := make([]byte, 100)
		if n, err = conn.Read(b); err != nil {
			panic(err)
		}
		fmt.Println("read data:", n, string(b))
	}()

	defer conn.Close()

	ch := make(chan os.Signal, 1)
	signal.Notify(ch, syscall.SIGHUP, syscall.SIGINT, syscall.SIGTERM, syscall.SIGQUIT)

	<-ch
}
```

因为之前读者问过一些问题，我计划编写一个从零开始实现的 easyio 系列教程，希望能够帮助一些小白通过实践来加深对这个主题的理解，如果你对这个系列感兴趣，请在下方留言或点赞。

决定的话，整个系列大概可以拆分成5～6篇文章(当然不收费～)。

看懂和会写出来是完全不一样的概念。理论与实践是相辅相成的，只有理解了理论并将其应用于实践中，我们才能真正掌握知识。

最后让这个世界充满爱～～～

源码地址:https://github.com/wuqinqiang/easyio





