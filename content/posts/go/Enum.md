---
https://github.com/wuqinqiangtitle: "iota 在 Go 中的使用 "
date: 2021-04-25T22:25:52+08:00 
toc: true 
tags:

- go
- enum

---

### 介绍

Go 语言实际上没有直接支持枚举的关键字。一般我们都是通过 `const` + `iota` 实现枚举的能力。

有人要问了，为什么一定要使用枚举呢？`stackoverflow` 上有一个高赞的回答，如下:
```
You should always use enums when a variable (especially a method parameter) can only take one out of a small set of possible values. Examples would be things like type constants (contract status: "permanent", "temp", "apprentice"), or flags ("execute now", "defer execution").

If you use enums instead of integers (or String codes), you increase compile-time checking and avoid errors from passing in invalid constants, and you document which values are legal to use.

BTW, overuse of enums might mean that your methods do too much (it's often better to have several separate methods, rather than one method that takes several flags which modify what it does), but if you have to use flags or type codes, enums are the way to go.

```
简单翻译一下， 两点很重要。
- 当一个变量(尤其是方法参数) 只能从一小部分可能的值中取出一个时，理应使用枚举。
例如类型常量(合同状态：永久、临时工、学徒)， 或者在做任务程序时，是立即执行还是延迟执行的标记。
- 如果使用枚举而不是整形，则会增加编译时的检查，避免错误无效值的传入，记录哪些值是合法使用的。



### 如何实现枚举
`iota` 是 Go 中预声明的一个特殊常量。它会被预声明为0，但是它的值在编译阶段并非是固定的，当预声明的 `iota` 出现在一个常量声明中的时候，它的值在第n个常量描述中的值为n(从0开始)。所以它只在同类型多个常量声明的情况下才显得有意义。

比如，大家都了解的电商，订单系统一定会涉及到订单状态的流转。那么这时候，我们一般可以这样做:

```go
package main

import "fmt"

type OrderStatus int

const (
	Cancelled OrderStatus = iota //订单已取消 0
	NoPay     OrderStatus = iota //未支付  1
	PendIng   OrderStatus = iota // 未发货 2
	Delivered OrderStatus = iota // 已发货 3
	Received  OrderStatus = iota // 已收货 4
)

func main() {
	fmt.Println(Cancelled, NoPay) // 打印:0,1
}

```

当然，这样看着好麻烦。其实，其他常量可以重复上一行 `iota` 表达式，我们可以改成这样。

```go
package main

import "fmt"

type OrderStatus int

const (
	Cancelled OrderStatus = iota //订单已取消 0
	NoPay                        //未支付 1
	PendIng                      // 未发货 2
	Delivered                    // 已发货 3
	Received                     // 已收货 4
)

func main() {
	fmt.Println(Cancelled, NoPay) // 打印:0,1
}

```
有人会用 0 的值来表示状态吗？一般都不会，我们想以1开头，那么可以这样。
```go
package main

import "fmt"

type OrderStatus int

const (
	Cancelled OrderStatus = iota+1 //订单已取消 1
	NoPay                        //未支付 2
	PendIng                      // 未发货 3
	Delivered                    // 已发货 4
	Received                     // 已收货 5
)

func main() {
	fmt.Println(Cancelled, NoPay) // 打印:1,2
}
```

我们还想在 `Delivered` 后跳过一个数字，才是 `Received` 的值,也就是 `Received=6`，那么可以借助 `_` 符号。

```go
package main

import "fmt"

type OrderStatus int

const (
	Cancelled OrderStatus = iota+1 //订单已取消 1
	NoPay                        //未支付 2
	PendIng                      // 未发货 3
	Delivered                    // 已发货 4
	_
	Received                     // 已收货 6
)

func main() {
	fmt.Println(Received) // 打印:6
}

```

顺着来可以，倒着当然也行。

```go
package main

import "fmt"

type OrderStatus int

const (
	Max = 5
)

const (
	Received  OrderStatus = Max - iota // 已收货  5
	Delivered                          // 已发货 4
	PendIng                            // 未发货 3
	NoPay                              //未支付 2
	Cancelled                          //订单已取消 1
)

func main() {
	fmt.Println(Received,Delivered) // 打印:5,4
}
```

你还可以使用位运算，比如在 go 源码中的包 `sync` 中的锁上面有这么一段代码。

```go
const (
    mutexLocked = 1 << iota  //1<<0
    mutexWoken               //1<<1
    mutexStarving            //1<<2
    mutexWaiterShift = iota  //3

)

func main() {
    fmt.Println("mutexLocked的值",mutexLocked) //打印：1
    fmt.Println("mutexWoken的值",mutexWoken) //打印：2
    fmt.Println("mutexStarving的值",mutexStarving) //打印：4
    fmt.Println("mutexWaiterShift的值",mutexWaiterShift) // 打印：3
}

```

可能有人平常是直接定义常量值或者用字符串来表示的。

比如，上面这些我完全可以用 `string` 来表示，我还真见过用字符串来表示订单状态的。
```go
package main

import "fmt"

const (
	Cancelled = "cancelled"
	NoPay     = "noPay"
	PendIng   = "pendIng"
	Delivered = "delivered"
	Received  = "received"
)

var OrderStatusMsg = map[string]string{
	Cancelled: "订单已取消",
	NoPay:     "未付款",
	PendIng:   "未发货",
	Delivered: "已发货",
	Received:  "已收货",
}

func main() {
	fmt.Println(OrderStatusMsg[Cancelled])
}
```
或者直接定义整形常量值。
```go
package main

import "fmt"

const (
	Cancelled = 1
	NoPay     = 2
	PendIng   = 3
	Delivered = 4
	Received  = 5
)

var OrderStatusMsg = map[int]string{
	Cancelled: "订单已取消",
	NoPay:     "未付款",
	PendIng:   "未发货",
	Delivered: "已发货",
	Received:  "已收货",
}

func main() {
	fmt.Println(OrderStatusMsg[Cancelled])
}

```
其实上述两种都可以，但是相比之下使用 `iota` 更有优势。
- 能保证一组常量的唯一性，人工定义的不能保证。
- 可以为一组动作分享同一种行为。
- 避免无效值。
- 提高代码阅读性以及维护。


### 延伸

按照上面我们所演示的，最后我们可以这样操作。
```go
package main

import (
	"fmt"
)

type OrderStatus int

const (
	Cancelled OrderStatus = iota + 1 //订单已取消 1
	NoPay                            //未支付 2
	PendIng                          // 未发货 3
	Delivered                        // 已发货 4
	Received                         // 已收货 5
)

//公共行为 赋予类型 String() 函数，方便打印值含义
func (order OrderStatus) String() string {
	return [...]string{"cancelled", "noPay", "pendIng", "delivered", "received"}[order-1]
}

//创建公共行为 赋予类型 int 函数 EnumIndex()
func (order OrderStatus) EnumIndex() int {
	return int(order)
}

func main() {
	var order OrderStatus = Received
	fmt.Println(order.String())    // 打印:received
	fmt.Println(order.EnumIndex()) // 打印:5
}

```

### 总结

这篇文章主要介绍了 `Golang` 中对 `iota` 的使用介绍，以及我们为什么要使用它。

不知道大家平常对于此类场景是用的什么招数，欢迎下方留言交流。





