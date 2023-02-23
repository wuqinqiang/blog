---
title: "nbio原理解析"
date: 2023-01-06T19:23:51+08:00 
toc: true 
tags :

- go
- net

---

上一篇文章我们介绍了evio，它是最早开源实现自定义go netpoll 实现，我们也提到它存在一些问题，这篇文章我们继续分析其他实现:nbio。

nbio项目也包含了在nbio之上构建的nbhttp，这个不在我们讨论范围。

老规矩，先来一段运行nbio程序代码，

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













