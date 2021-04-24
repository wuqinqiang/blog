---
title: "在Go中高频犯过的错 "
date: 2021-04-21T23:54:52+08:00
toc: true
tags :

- go
- mistake

---

### 在迭代器变量上使用goroutine

我们来看一段代码

```go
package main

import (
	"fmt"
	"sync"
)

type User struct {
	userId int
}

func main() {
	var userList []User
	for i := 0; i < 10; i++ {
		userList = append(userList, User{userId: i})
	}

	var wg sync.WaitGroup
	for i, _ := range userList {
		wg.Add(1)
		go func() {
			defer wg.Done()
			Do(userList[i])
		}()
	}
	wg.Wait()
}

func Do(user User) {
	fmt.Printf("用户id:%v\n", user.userId)
}

```

一个很简单的利用 `sync.waitGroup` 做任务编排的场景，看一下好像没啥问题，运行看看结果。

![image](https://image.syst.top/image/mistake/mistake-1.png)

啥情况，为啥不是1-9(当然不是顺序的)。

原因很简单，循环器中的 i 实际上是一个单变量，`go func`  里的闭包只绑定在一个变量上，
每个 `goroutine` 可能要等到循环结束才真正的运行，这时候运行的 i 值大概率就是 9了。
大概率的意思是没人能保证这个过程，有的只是手段。

正确的做法

```go
package main

import (
	"fmt"
	"sync"
)

type User struct {
	userId int
}

func main() {
	var userList []User
	for i := 0; i < 10; i++ {
		userList = append(userList, User{userId: i})
	}

	var wg sync.WaitGroup
	for i, _ := range userList {
		wg.Add(1)
		go func(user User) {
			defer wg.Done()
			Do(user)
		}(userList[i])
	}
	wg.Wait()
}

func Do(user User) {
	fmt.Printf("用户id:%v\n", user.userId)
}
```

通过将 `user`  作为一个参数传入闭包中，user 每次迭代都会被求值，
并放置在 `goroutine` 的堆栈中，因此每个用户列表切片元素最终都会被执行打印。

或者这样,道理是一样的
```go
for i, _ := range userList {
		wg.Add(1)
		index:=i
		go func() {
			defer wg.Done()
			Do(userList[index])
		}()
	}
```

### WaitGroup


上面说到 `sync.waitGroup`，它也有犯错的地方，之前帮同事看代码的时候也发现这个错误。
我把上面的例子稍微改动一下
```go
package main

import (
	"errors"
	"github.com/prometheus/common/log"
	"sync"
)

type User struct {
	userId int
}

func main() {
	var userList []User
	for i := 0; i < 10; i++ {
		userList = append(userList, User{userId: i})
	}

	var wg sync.WaitGroup
	for i, _ := range userList {
		wg.Add(1)
		go func(item int) {
			_, err := Do(userList[item])
			if err != nil {
				log.Infof("err message:%v\n", err)
				return
			}
			wg.Done()
		}(i)
	}
	wg.Wait()
	
	// 处理其他事务
}

func Do(user User) (string, error) {
	// 处理杂七杂八的业务....
	if user.userId == 9 {
		// 此人是非法用户
		return "失败", errors.New("非法用户")
	}
	return "成功", nil
}
```

发现问题严重性了吗？

当用户id等于9的时候，`err !=nil` 直接 `return` 了，导致 `waitGroup` 计数器根本没机会减1，
最终 `wait` 阻塞在那，多么可怕的bug。

所以一般，不，是绝大多数的场景下，我们都必须这样:
```go
go func(item int) {
			defer wg.Done()
			//....业务代码
			//....业务代码
			_, err := Do(userList[item])
			if err != nil {
				log.Infof("err message:%v\n", err)
				return
			}
		}(i)
```
### goroutine 泄露

```go
package main

import (
	"context"
	"github.com/prometheus/common/log"
	"time"
)

func main() {
	ch := make(chan struct{})
	c, cancel := context.WithCancel(context.Background())

	go func() {
		ch<- struct{}{}
		time.Sleep(3 * time.Second)
		cancel()
	}()

	for {
		select {
		case res := <-ch:
			log.Infof("item:%v", res)
		case <-c.Done():
			log.Infof("结束")
			break
		}
	}
}

```
我们的本意是三秒后终止 `for` 循环，我们这里用了 `break`。
`break` 与 `select` 有关，和 `for` 语句无关！
可以借助带标签的 break，比如下面这种 
`return`。
```go
package main

import (
	"context"
	"github.com/prometheus/common/log"
	"time"
)

func main() {
	ch := make(chan struct{})
	c, can := context.WithCancel(context.Background())

	go func() {
		ch <- struct{}{}
		time.Sleep(2 * time.Second)
		can()
	}()

loop:
	for {
		select {
		case res := <-ch:
			log.Infof("item:%v", res)
		case <-c.Done():
			log.Infof("结束")
			break loop
		}
	}
}
```

或者如果不需要往下走的直接 `return`。


### 野生 goroutine

我不知道你们公司是咋么处理异步操作的，是下面这样的吗
```go
func main() {
	// doSomething
	go func() {
		// doSomething
	}()
}
```
我们为了防止程序中出现不可预知的 `panic`，导致程序直接挂掉，都会加入 `recover`，比如这样
```go
func main() {
	defer func() {
		if err := recover(); err != nil {
			fmt.Printf("%v\n", err)
		}
	}()
	panic("处理失败")
}
```

但是如果这时候我们直接开启一个 `goroutine`，在这个 `goroutine` 里面发生了  `panic`，比如这样

```go
func main() {
	defer func() {
		if err := recover(); err != nil {
			fmt.Printf("%v\n", err)
		}
	}()
	go func() {
		panic("处理失败")
	}()

	time.Sleep(2 * time.Second)
}
```
此时最外层的 `recover` 并不能捕获，程序会直接挂掉，就像下面这样。
![image](https://image.syst.top/image/mistake/mistake-2.png)

但是你总不能每次都在开启一个新的 `goroutine` 像下面这样写,
```go
func main() {
	defer func() {
		if err := recover(); err != nil {
			fmt.Printf("%v\n", err)
		}
	}()

	// func1
	go func() {
		defer func() {
			if err := recover(); err != nil {
				fmt.Printf("%v\n", err)
			}
		}()
		panic("错误失败")
	}()

	// func2
	go func() {
		defer func() {
			if err := recover(); err != nil {
				fmt.Printf("%v\n", err)
			}
		}()
		panic("请求错误")
	}()

	time.Sleep(2 * time.Second)
}
```
所以基本上大家都会包一层。比如这样写,
```go
package main

import (
	"fmt"
	"time"
)

func main() {
	defer func() {
		if err := recover(); err != nil {
			fmt.Printf("%v\n", err)
		}
	}()

	// func1
	Go(func() {
		panic("错误失败")
	})

	// func2
	Go(func() {
		panic("请求错误")
	})

	time.Sleep(2 * time.Second)
}

func Go(fn func()) {
	go RunSafe(fn)
}

func RunSafe(fn func()) {
	defer func() {
		if err := recover(); err != nil {
			fmt.Printf("错误:%v\n", err)
		}
	}()
	fn()
}
```

当然我这里只是简单都打印一些日志信息，一般还会带上堆栈都信息。


### channel 

`channel` 在 `go` 中的地位实在太高了，各大开源项目到处都是 `channel` 的影子，
以至于你在工业级的项目 issues 中搜索 `channel` ，能看到很多的 `bug`，
比如 etcd 这个 `issues`,
![image](https://image.syst.top/image/mistake/mistake-3.png)

一个往已关闭的 `channel` 中发送数据数据引发的 `panic`,等等很多。
这个故事告诉我们，否管大不大佬，改写的 `bug` 还是会写。

`channel` 除了上述高频出现的错误，还有以下几点:

直接关闭一个 nil 值 channel 会引发 panic
```go
package main

func main() {
  var ch chan struct{}
  close(ch)
}

```

关闭一个已关闭的 channel 会引发 panic。
```go
package main

func main() {
  ch := make(chan struct{})
  close(ch)
  close(ch)
}
```

还有一点，有时候使用 channel 不小心会导致 goroutine 泄露，比如下面这种情况,

```go
package main

import (
	"context"
	"fmt"
	"time"
)

func main() {
	ch := make(chan struct{})
	cx, _ := context.WithTimeout(context.Background(), time.Second)
	go func() {
		time.Sleep(2 * time.Second)
		ch <- struct{}{}
		fmt.Println("goroutine 结束")
	}()

	select {
	case <-ch:
		fmt.Println("res")
	case <-cx.Done():
		fmt.Println("timeout")
	}
	time.Sleep(5 * time.Second)
}

```

启动一个 goroutine 去处理业务，业务需要执行2秒，超时时间是1秒，
会导致 channel 从未被读取，
我们知道没有缓冲的 channel 必须等发送方和接收方都准备好才能操作。
此时子 `goroutine` 会被永久阻塞在  `ch <- struct{}` 这行代码，除非程序结束，而这就是 goroutine 泄露。

解决这个也很简单，把无缓冲的 channel 改成缓冲为1。 
 






















