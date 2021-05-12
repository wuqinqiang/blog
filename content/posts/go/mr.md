---
title: "业务中，经常需要的工具(1)"
date: 2021-04-21T23:54:52+08:00
toc: true
tags :

- go
- mistake

---

### 业务场景
在做任务开发的时候，你们一定会碰到以下场景:

场景1：在调用外部第三方接口的时候， 某些第三方我严重怀疑他们自己开发的接口从来没有用到自己的业务上，
然后你就会发现，一个需求你需要调用不同的接口，做数据组装。

场景2:一个应用首页可能依托于很多服务。那就涉及到在加载页面时需要同时请求多个服务的接口。
这一步往往是由后端统一调用组装数据再返回给前端，也就是所谓的 BFF(Backend For Frontend) 层。
调用这些服务往往没有前后的依赖关系，可以同时调用。

针对以上两种场景，假设在没有强依赖关系下，选择串行话调用，那么总耗时即:
```
time=s1+s2+....sn
```
按照当代秒入百万的有为青年，这么长时间早就把你祖宗十八代问候了一遍。

为了伟大的KPI，我们往往会选择并发的调用这些依赖接口。那么总耗时就是:
```
time=max(s1,s2,s3.....,sn)
```
当然开始堆业务的时候可以先串行化，等到上面的人着急的时候，亮出绝招。
这样，年底 `PPT` 就可以加上浓重的一笔流水账:为业务某个接口提高百分之XXX性能，间接产生XXX价值。

当然这一切的前提是，做老板不懂技术，做技术"懂"你。

言归正传,

如果修改成并发调用，你可能会这么写，
```go
package main

import (
	"fmt"
	"sync"
	"time"
)

func main() {
	var wg sync.WaitGroup
	wg.Add(2)

	var userInfo *User
	var productList []Product
	
	go func() {
		defer wg.Done()
		userInfo, _ = getUser()
	}()

	go func() {
		defer wg.Done()
		productList, _ = getProductList()
	}()
	wg.Wait()
	fmt.Printf("用户信息:%+v\n", userInfo)
	fmt.Printf("商品信息:%+v\n", productList)
}


/********用户服务**********/

type User struct {
	Name string
	Age  uint8
}

func getUser() (*User, error) {
	time.Sleep(500 * time.Millisecond)
	var u User
	u.Name = "wuqinqiang"
	u.Age = 18
	return &u, nil
}

/********商品服务**********/

type Product struct {
	Title string
	Price uint32
}

func getProductList() ([]Product, error) {
	time.Sleep(400 * time.Millisecond)
	var list []Product
	list = append(list, Product{
		Title: "SHib",
		Price: 10,
	})
	return list, nil
}
```

先不管其他问题。从实现上来说，需要多少服务，你会开多少个 `G`，利用 `sync.WaitGroup` 的特性，
实现并发编排任务的效果。

好像，问题不大。

但是随着代号 `996` 业务场景的增加，你会发现，好多模块都有相似的功能，只是对应的业务场景不同而已。

那么我们能不能抽像出一套针对此业务场景的工具，而把具体业务实现交给业务方。

安排。

### 使用
本着不重复造轮子的原则，去搜了下开源项目，最终看上了 `go-zero` 里面的一个工具 `mapreduce`。
从文件名我们能看出来是什么了，可以自行 `Google` 这个名词。

使用很简单。我们通过它改造一下上面的代码:
```go
package main

import (
	"fmt"
	"github.com/tal-tech/go-zero/core/mr"
	"time"
)

func main() {
	var userInfo *User
	var productList []Product
	_ = mr.Finish(func() (err error) {
		userInfo, err = getUser()
		return err
	}, func() (err error) {
		productList, err = getProductList()
		return err
	})
	fmt.Printf("用户信息:%+v\n", userInfo)
	fmt.Printf("商品信息:%+v\n", productList)
}
```
```
用户信息:&{Name:wuqinqiang Age:18}
商品信息:[{Title:SHib Price:10}]
```

是不是舒服多了。

但是这里还需要注意一点，假设你调用的其中一个服务错误，并且你 `return err` 对应的错误，那么其他调用的服务会被取消。
比如我们修改 getProductList 直接响应错误。

```go
func getProductList() ([]Product, error) {
	return nil, errors.New("test error")
}
//打印
用户信息:<nil>
商品信息:[]
```
那么最终打印的时候连用户信息都会为空，因为出现一个服务错误，用户服务请求被取消了。

一般情况下，在请求服务错误的时候我们会有保底操作，一个服务错误不能影响其他请求的结果。
所以在使用的时候具体处理取决于业务场景。

### 源码
既然用了，那么就追下源码吧。

```go
func Finish(fns ...func() error) error {
	if len(fns) == 0 {
		return nil
	}

	return MapReduceVoid(func(source chan<- interface{}) {
		for _, fn := range fns {
			source <- fn
		}
	}, func(item interface{}, writer Writer, cancel func(error)) {
		fn := item.(func() error)
		if err := fn(); err != nil {
			cancel(err)
		}
	}, func(pipe <-chan interface{}, cancel func(error)) {
		drain(pipe)
	}, WithWorkers(len(fns)))
}
```
```go
func MapReduceVoid(generator GenerateFunc, mapper MapperFunc, reducer VoidReducerFunc, opts ...Option) error {
	_, err := MapReduce(generator, mapper, func(input <-chan interface{}, writer Writer, cancel func(error)) {
		reducer(input, cancel)
		drain(input)
		// We need to write a placeholder to let MapReduce to continue on reducer done,
		// otherwise, all goroutines are waiting. The placeholder will be discarded by MapReduce.
		writer.Write(lang.Placeholder)
	}, opts...)
	return err
}
```

对于 `MapReduceVoid`函数，主要查看三个闭包参数。
- 第一个 `GenerateFunc` 用于生产数据。
- `MapperFunc` 读取生产出的数据，进行处理。
- `VoidReducerFunc` 这里表示不对 `mapper` 后的数据做聚合返回。所以这个闭包在此操作几乎0作用。

```go
func MapReduce(generate GenerateFunc, mapper MapperFunc, reducer ReducerFunc, opts ...Option) (interface{}, error) {
	source := buildSource(generate) 
	return MapReduceWithSource(source, mapper, reducer, opts...)
}

func buildSource(generate GenerateFunc) chan interface{} {
	source := make(chan interface{})// 创建无缓冲通道
	threading.GoSafe(func() {
		defer close(source)
		generate(source) //开始生产数据
	})

	return source //返回无缓冲通道
}
```
`buildSource`函数中，返回一个无缓冲的通道。并开启一个 `G` 运行 `generate(source)`，往无缓冲通道塞数据。 这个`generate(source)` 不就是一开始 `Finish` 传递的第一个闭包参数。
```go
return MapReduceVoid(func(source chan<- interface{}) {
	// 就这个
		for _, fn := range fns {
			source <- fn
		}
	})
```

然后查看 `MapReduceWithSource` 函数,

```go
func MapReduceWithSource(source <-chan interface{}, mapper MapperFunc, reducer ReducerFunc,
	opts ...Option) (interface{}, error) {
	options := buildOptions(opts...)
	//任务执行结束通知信号
	output := make(chan interface{})
    //将mapper处理完的数据写入collector
    collector := make(chan interface{}, options.workers)
    // 取消操作信号
	done := syncx.NewDoneChan()
	writer := newGuardedWriter(output, done.Done())
	var closeOnce sync.Once
	var retErr errorx.AtomicError
	finish := func() {
		closeOnce.Do(func() {
			done.Close()
			close(output)
		})
	}
	cancel := once(func(err error) {
		if err != nil {
			retErr.Set(err)
		} else {
			retErr.Set(ErrCancelWithNil)
		}

		drain(source)
		finish()
	})

	go func() {
		defer func() {
			if r := recover(); r != nil {
				cancel(fmt.Errorf("%v", r))
			} else {
				finish()
			}
		}()
		reducer(collector, writer, cancel)
		drain(collector)
	}()
	// 真正从生成器通道取数据执行Mapper
	go executeMappers(func(item interface{}, w Writer) {
		mapper(item, w, cancel)
	}, source, collector, done.Done(), options.workers)

	value, ok := <-output
	if err := retErr.Load(); err != nil {
		return nil, err
	} else if ok {
		return value, nil
	} else {
		return nil, ErrReduceNoOutput
	}
}
```
这段代码挺长的，我们说下核心的点。我们看到使用一个`G` 调用 `executeMappers` 方法。
```go
go executeMappers(func(item interface{}, w Writer) {
		mapper(item, w, cancel)
	}, source, collector, done.Done(), options.workers)
```

```go
func executeMappers(mapper MapFunc, input <-chan interface{}, collector chan<- interface{},
	done <-chan lang.PlaceholderType, workers int) {
	var wg sync.WaitGroup
	defer func() {
		// 等待所有任务全部执行完毕
		wg.Wait()
		// 关闭通道
		close(collector)
	}()
   //根据指定数量创建 worker池
	pool := make(chan lang.PlaceholderType, workers) 
	writer := newGuardedWriter(collector, done)
	for {
		select {
		case <-done:
			return
		case pool <- lang.Placeholder:
			// 从buildSource() 返回的无缓冲通道取数据
			item, ok := <-input 
			// 当通道关闭，结束
			if !ok {
				<-pool
				return
			}

			wg.Add(1)
			// better to safely run caller defined method
			threading.GoSafe(func() {
				defer func() {
					wg.Done()
					<-pool
				}()
				//真正运行闭包函数的地方
               // func(item interface{}, w Writer) {
               //    mapper(item, w, cancel)
               //    }
				mapper(item, writer)
			})
		}
	}
}
```

具体的逻辑已备注，代码很容易懂。

一旦 `executeMappers` 函数返回，关闭 `collector` 通道，那么执行 `reducer` 不再阻塞。 
```go
go func() {
		defer func() {
			if r := recover(); r != nil {
				cancel(fmt.Errorf("%v", r))
			} else {
				finish()
			}
		}()
		reducer(collector, writer, cancel)
		//这里
		drain(collector)
	}()
```
这里的 `reducer(collector, writer, cancel)` 其实就是从 `MapReduceVoid` 传递的第三个闭包函数。 

```go
func MapReduceVoid(generator GenerateFunc, mapper MapperFunc, reducer VoidReducerFunc, opts ...Option) error {
	_, err := MapReduce(generator, mapper, func(input <-chan interface{}, writer Writer, cancel func(error)) {
		reducer(input, cancel)
		//这里
		drain(input)
		// We need to write a placeholder to let MapReduce to continue on reducer done,
		// otherwise, all goroutines are waiting. The placeholder will be discarded by MapReduce.
		writer.Write(lang.Placeholder)
	}, opts...)
	return err
}
```

然后这个闭包函数又执行了 `reducer(input, cancel)`，这里的 `reducer` 就是我们一开始解释过的 `VoidReducerFunc`，从 `Finish() 而来`。

![image](https://image.syst.top/image/mr/mr.png)

等等,看到上面三个地方的 `drain(input)`了吗？
```go
// drain drains the channel.
func drain(channel <-chan interface{}) {
	// drain the channel
	for range channel {
	}
}
```
其实就是一个排空 `channel` 的操作，但是三个地方都对同一个 `channel`,也是让我费解。

还有更重要的一点。
```go
go func() {
		defer func() {
			if r := recover(); r != nil {
				cancel(fmt.Errorf("%v", r))
			} else {
				finish()
			}
		}()
		reducer(collector, writer, cancel)
		drain(collector)
	}()
```
上面的代码，假如执行 `reducer`，`writer` 写入引发 `panic`,那么`drain(collector)` 会直接卡住。

不过作者已经修复了这个问题，直接把 `drain(collector)` 放入到 `defer`。
![image](https://image.syst.top/image/mr/mr-2.png)

具体 issues。

到这里，关于 `Finish` 的源码也就结束了。感兴趣的可以看看其他源码。

很喜欢 `go-zero` 里的一些工具，但是往往用的一些工具并不独立，
依赖于其他文件包，导致明明只想使用其中一个工具却需要安装整个包。
所以最终的结果就是扒源码，创建无依赖库工具集，遵循 `MIT` 即可。





 






















