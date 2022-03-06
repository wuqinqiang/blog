---
title: "一个用go实现的有限状态机 "
date: 2022-03-06T17:37:22+08:00 
toc: true 
tags:

- go
- fsm

---





### easyfsm

之前看过新亮老哥的go-fsm-order，感觉还不错。最近在迁移项目的时候，发现有多处业务存在一些状态的流转，所以就基于go-fsm-order做了重改，让它可以在不同的业务场景下使用。

为什么不使用looplab/fsm，star挺多的啊。

不是特别喜欢，每次实例化fsm都需要重新传递对应events(虽然我们可以统一封装)，我更期望在项目启动时把此项目涉及到不同业务状态机流转注册到fsm，对应:不同业务->[状态]->[事件]->处理事件主体(包含handler、params、hooks、observers等)。

当你开始进行状态流转时，只需要

```go
fsm:=NewFsm("业务名称","当前状态")
currentState,err:=fsm.Call("事件名称","对应事件所需参数可选项")
```

为什么需要区分业务？

因为绝大多数业务的状态值都是从数据库中获取的，比如订单表的订单状态，商品表中的商品状态，有可能值是相同的。

同一个业务同一属性对应状态值表达单一，不同业务下属性状态可能会出现值相同，但所表达的含义是不同的。

整体设计:

![fsm](https://cdn.syst.top/easyfsm.png)

简单解释一下：

- 业务:比如有商品状态业务、订单状态业务.....
- 状态：订单待付款、待发货....
- 事件：对应状态仅可达事件集合。比如待付款状态的可达事件仅有:支付事件和取消事件(取决于自己的业务)
- 执行事件主体：执行自定义的事件函数,如果有需要，还可以自定义执行事件前后hook，事件订阅者(比如支付事件发生后，异步通知用户等)



### 使用姿势

```go
go get -u  github.com/wuqinqiang/easyfsm
```



首先自定义业务、状态、事件。

```go
package main

import (
	"fmt"
	"github.com/wuqinqiang/easyfsm"
)

var (
	// 业务
	businessName easyfsm.BusinessName = "order"

	// 对应状态
	initState easyfsm.State = 1 // 初始化
	paidState easyfsm.State = 2 // 已付款
	canceled  easyfsm.State = 3 // 已取消

	//对应事件
	paymentOrderEvent easyfsm.EventName = "paymentOrderEvent"
	cancelOrderEvent  easyfsm.EventName = "cancelOrderEvent"
)
```



自定义事件主体，

```go
type (
  // 订单业务所需参数
	orderParam struct {
		OrderNo string
    //....
	}
  
  // 支付订单事件
	entity := easyfsm.NewEventEntity(paymentOrderEvent,
		func(opt *easyfsm.Param) (easyfsm.State, error) {
			param, ok := opt.Data.(orderParam)
			if !ok {
				panic("param err")
			}
			fmt.Printf("param:%+v\n", param)
			// 处理核心业务
			return paidState, nil
		})

	// 取消订单事件
	cancelEntity := easyfsm.NewEventEntity(cancelOrderEvent,
		func(opt *easyfsm.Param) (easyfsm.State, error) {
			// 处理核心业务
			param, ok := opt.Data.(orderParam)
			if !ok {
				panic("param err")
			}
			fmt.Printf("param:%+v\n", param)
			return canceled, nil
		})


  
```

注册到easyfsm。

```go
	// 注册到fsm
	easyfsm.RegisterStateMachine(businessName,
		initState,
		entity, cancelEntity)
```

开始使用。

````go
  // 正常操作

  // 第一步根据业务，以及当前状态生成fsm
	fsm := easyfsm.NewFSM(businessName, easyfsm.State(1))

	// 第二步 调用具体事件
	currentState, err := fsm.Call(cancelOrderEvent,
		easyfsm.WithData(orderParam{OrderNo: "wuqinqiang050@gmail.com"}))

	fmt.Printf("[Success]call cancelOrderEvent err:%v\n", err)
	fmt.Printf("[Success]call cancelOrderEvent state:%v\n", currentState)
 
  // 结束
````

示例代码如下，

```go
package main

import (
	"fmt"
	"github.com/wuqinqiang/easyfsm"
)

var (
	// 业务
	businessName easyfsm.BusinessName = "order"

	// 对应状态
	initState easyfsm.State = 1 // 初始化
	paidState easyfsm.State = 2 // 已付款
	canceled  easyfsm.State = 3 // 已取消

	//对应事件
	paymentOrderEventName easyfsm.EventName = "paymentOrderEventName"
	cancelOrderEventName  easyfsm.EventName = "cancelOrderEventName"
)

type (
	orderParam struct {
		OrderNo string
	}
)

func init() {
	// 支付订单事件
	entity := easyfsm.NewEventEntity(paymentOrderEventName,
		func(opt *easyfsm.Param) (easyfsm.State, error) {
			param, ok := opt.Data.(orderParam)
			if !ok {
				panic("param err")
			}
			fmt.Printf("param:%+v\n", param)
			// 处理核心业务
			return paidState, nil
		})

	// 取消订单事件
	cancelEntity := easyfsm.NewEventEntity(cancelOrderEventName,
		func(opt *easyfsm.Param) (easyfsm.State, error) {
			// 处理核心业务
			param, ok := opt.Data.(orderParam)
			if !ok {
				panic("param err")
			}
			fmt.Printf("param:%+v\n", param)
			return canceled, nil
		})

	// 注册订单状态机
	easyfsm.RegisterStateMachine(businessName,
		initState,
		entity, cancelEntity)
}

func main() {

	// 正常操作

	// 第一步根据业务，以及当前状态生成fsm
	fsm := easyfsm.NewFSM(businessName, initState)

	// 第二步 调用具体
	currentState, err := fsm.Call(cancelOrderEventName,
		easyfsm.WithData(orderParam{OrderNo: "wuqinqiang050@gmail.com"}))

	fmt.Printf("[Success]call cancelOrderEventName err:%v\n", err)
	fmt.Printf("[Success]call cancelOrderEventName state:%v\n", currentState)

	//异常情况1，没有定义goods业务
	fsm = easyfsm.NewFSM("goods", paidState)
	currentState, err = fsm.Call(cancelOrderEventName,
		easyfsm.WithData(orderParam{OrderNo: "wuqinqiang050@gmail.com"}))
	fmt.Printf("[UnKnowBusiness]faild :%v\n", err)
	fmt.Printf("[UnKnowBusiness]faild state:%v\n", currentState)

	//异常情况1,没有定义状态:2
	fsm = easyfsm.NewFSM(businessName, easyfsm.State(2))
	currentState, err = fsm.Call(cancelOrderEventName,
		easyfsm.WithData(orderParam{OrderNo: "wuqinqiang050@gmail.com"}))
	fmt.Printf("[UnKnowState]faild :%v\n", err)
	fmt.Printf("[UnKnowState]faild state:%v\n", currentState)

	//异常情况2:没有定义状态1对应的发货事件
	fsm = easyfsm.NewFSM(businessName, initState)
	currentState, err = fsm.Call("shippingEvent",
		easyfsm.WithData(orderParam{OrderNo: "wuqinqiang050@gmail.com"}))
	fmt.Printf("[UnKnowEvent]faild :%v\n", err)
	fmt.Printf("[UnKnowEvent]faild state:%v\n", currentState)

}
```



### Hook

如果想在处理事件函数的前后执行一些hook，或者在事件执行完毕，异步执行一些其他业务，easyfsm定义了这两个接口，

```go
type (
	EventObserver interface {
		Receive(opt *Param)
	}

	EventHook interface {
		Before(opt *Param)
		After(opt Param, state State, err error)
	}
)
```

我们可以实现这两个接口，

```go
type (
	NotifyExample struct {
	}
	HookExample struct {
	}
)

func (h HookExample) Before(opt *easyfsm.Param) {
	println("事件执行前")
}

func (h HookExample) After(opt easyfsm.Param, state easyfsm.State, err error) {
	println("事件执行后")
}

func (o NotifyExample) Receive(opt *easyfsm.Param) {
	println("接收到事件变动,发送消息")
}
```

完整代码：

```go
package main

import (
	"fmt"
	"github.com/wuqinqiang/easyfsm"
	"time"
)

var (
	// 业务
	businessName easyfsm.BusinessName = "order"

	// 对应状态
	initState easyfsm.State = 1 // 初始化
	paidState easyfsm.State = 2 // 已付款
	canceled  easyfsm.State = 3 // 已取消

	//对应事件
	paymentOrderEventName easyfsm.EventName = "paymentOrderEventName"
	cancelOrderEventName  easyfsm.EventName = "cancelOrderEventName"
)

type (
	orderParam struct {
		OrderNo string
	}
)

var (
	_ easyfsm.EventObserver = (*NotifyExample)(nil)
	_ easyfsm.EventHook     = (*HookExample)(nil)
)

type (
	NotifyExample struct {
	}
	HookExample struct {
	}
)

func (h HookExample) Before(opt *easyfsm.Param) {
	println("事件执行前")
}

func (h HookExample) After(opt easyfsm.Param, state easyfsm.State, err error) {
	println("事件执行后")
}

func (o NotifyExample) Receive(opt *easyfsm.Param) {
	println("接收到事件变动,发送消息")
}

func init() {
	// 支付订单事件
	entity := easyfsm.NewEventEntity(paymentOrderEventName,
		func(opt *easyfsm.Param) (easyfsm.State, error) {
			param, ok := opt.Data.(orderParam)
			if !ok {
				panic("param err")
			}
			fmt.Printf("param:%+v\n", param)
			// 处理核心业务
			return paidState, nil
		}, easyfsm.WithHook(HookExample{}), easyfsm.WithObservers(NotifyExample{}))

	// 取消订单事件
	cancelEntity := easyfsm.NewEventEntity(cancelOrderEventName,
		func(opt *easyfsm.Param) (easyfsm.State, error) {
			// 处理核心业务
			param, ok := opt.Data.(orderParam)
			if !ok {
				panic("param err")
			}
			fmt.Printf("param:%+v\n", param)
			return canceled, nil
		}, easyfsm.WithHook(HookExample{}))

	// 注册订单状态机
	easyfsm.RegisterStateMachine(businessName,
		initState,
		entity, cancelEntity)
}

func main() {

	// 正常操作

	// 第一步根据业务，以及当前状态生成fsm
	fsm := easyfsm.NewFSM(businessName, initState)

	// 第二步 调用具体
	currentState, err := fsm.Call(paymentOrderEventName,
		easyfsm.WithData(orderParam{OrderNo: "wuqinqiang050@gmail.com"}))

	fmt.Printf("[Success]call paymentOrderEventName err:%v\n", err)
	fmt.Printf("[Success]call paymentOrderEventName state:%v\n", currentState)
	time.Sleep(2 * time.Second)
}
```



### 结束

如果有其他不一样的需求，欢迎大家在issue留言。

GitHub:[easyfsm](https://github.com/wuqinqiang/easyfsm)