---
title: "原来sync.Once还能这么用"
date: 2021-04-25T22:25:52+08:00 
toc: true 
tags:

- go
- enum


---



### 介绍

`sync.Once`估计大家都不陌生，官方介绍中，

> Once is an object that will perform exactly one action

正是因为这个特性，`Once`常常被用于单例对象的初始化场景。

也正是因为这个特性，其实它还能做一些其他的事情。



### 缓存击穿

日常背诵八股文，我相信你们对**缓存击穿**这个词特别熟悉。

**缓存击穿**一般待指热点`key`缓存失效(到期|删了)，同一时刻大量对热点`key`的并发请求。缓存找不到数据，所有请求都打入到`DB`层。此时，身为开发的你，明天和意外就不知道哪个先到了。

为了防止这种情况发生，针对相同`key`的请求，只需要一个请求(A)到达`DB`层取数据，其他请求等待`A`通知就行了。

就像这样，

![缓存击穿](https://cdn.syst.top/%E7%BC%93%E5%AD%98%E5%87%BB%E7%A9%BF.png)

​                                 图片来源:[caching](https://medium.com/codex/caching-system-stability-766bf5fff69f)



### singleflight

`Go`里有很多防缓存击穿的工具，比如`singleflight`库。

```go
type call struct {
	wg sync.WaitGroup
	val interface{}
	err error
	forgotten bool
	//.....省略部分字段
}

type Group struct {
	mu sync.Mutex       
	m  map[string]*call 
}
```

![](https://cdn.syst.top/once-1.png)

通过上面简单的代码大概能看出，其实就是对`key`做了缓存。

把一个`key`对应`call`结构存储在`map`中。保证只有一个`key`真正执行`fn()`服务 ，其他请求则通过`sync.waitGroup`的`wait`等待结果。

至于`g.docall(c,key,fn)`，

![](https://cdn.syst.top/once-2.png)

当带着全村人希望的那个请求，获取到数据，给对应`key`的`call`赋值，最终执行`done`，通知等待这个`key`全村的村民获取数据。

代码并不复杂。



### **自定义**singleflight

我们也可以实现一个简易版本的。

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

type CacheEntry struct {
	data []byte
	err  error
	wait chan struct{}
}

type OrderSever struct {
	cache map[string]*CacheEntry
	mutex sync.Mutex
}

func (order *OrderSever) Query(key string) ([]byte, error) {
	order.mutex.Lock()
	if order.cache == nil {
		order.cache = make(map[string]*CacheEntry)
	}
  
  //已经有其他兄弟请求了，你等等
	if entry, ok := order.cache[key]; ok {
		order.mutex.Unlock()
		<-entry.wait
		return entry.data, entry.err
	}
	entry := &CacheEntry{
		data: make([]byte, 0),
		wait: make(chan struct{}),
	}
	order.cache[key] = entry
	order.mutex.Unlock()
  // 请求数据
	entry.data, entry.err = getOrder()
  // 请求数据完毕，通知其他兄弟可以拿数据了。
	close(entry.wait)
	return entry.data, nil
}

//外部服务
func getOrder() ([]byte, error) {
	time.Sleep(50 * time.Millisecond)
	return []byte("hello world"), nil
}
```

代码整体不难，主要的点在于我们是通过通道来实现通知自家兄弟取数据。

最后，让我们使用`Once`来达到同样的效果，不然标题不白起了嘛。

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

type CacheEntry struct {
	data []byte
	err  error
	once *sync.Once
}

type OrderSever struct {
	cache map[string]*CacheEntry
	mutex sync.Mutex
}

func (order *OrderSever) Query(key string) ([]byte, error) {
	order.mutex.Lock()
	if order.cache == nil {
		order.cache = make(map[string]*CacheEntry)
	}
	entry, ok := order.cache[key]
  
  // 找不到就初始化一个
	if !ok {
		entry = &CacheEntry{
			data: make([]byte, 0),
			err:  nil,
			once: new(sync.Once),
		}
		order.cache[key] = entry
	}
	order.mutex.Unlock()
  
  // 我只执行一次。
	entry.once.Do(func() {
		entry.data, entry.err = getOrder()
	})
  //数据被赋值了，返回
	return entry.data, entry.err
}

func getOrder() ([]byte, error) {
	time.Sleep(50 * time.Millisecond)
	return []byte("hello world"), nil
}
```

上面核心代码都写出来了，实际开发中需要对请求资源做一些超时控制等操作。



### 总结

平常对`Once`的使用只停留在初始化工作上，而弱化了它的使用场景。对于其他工具也是一个道理，这就需要去积累和挖掘了。



### **附录**

[1]https://medium.com/codex/caching-system-stability-766bf5fff69f

https://blog.chuie.io/posts/synconce/
