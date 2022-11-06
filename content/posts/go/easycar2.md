---
title: "easycar更新日记"
date: 2022-11-06T10:11:22+08:00 
toc: true 
tags:

- go
- easycar
- distributed transaction 
---

#### 开篇

又拖更了一个多月，在思考人生的意义。

上一次介绍了新开发的一个分布式的事务框架easycar，这篇就当是对easycar的更新日记了。

**服务注册与发现**

由于easycar底层基于gRPC, 通过自定义Resolver接口还是很容易实现的。

**负载均衡**

客户端负载均衡，目前支持

- `round-robin`

- `random`
- `power of 2 random choice`
- `consistent hash`
- `ip-hash`
- `least-load`

架构图

![easycar](https://cdn.syst.top/easycar2.jpg)

上次提到，参与的一组分布式事务可能部分操作存在先后顺序的问题。

我举了个例子，我们需要保证必须先执行account扣减余额和stock扣减库存服务成功后，才能创建订单order的服务。同时account和stock服务并不需要保证他们的执行顺序。下图，

![global](https://cdn.syst.top/servers.png)以这个例子，那么实际在easycar中整个流程，

 **成功**

![success](https://cdn.syst.top/success2.png)

 **失败**

![failed](https://cdn.syst.top/failed2.png)

在easycar中，client负责和easycar(TC)端交互。主要负责注册分支，触发执行分布式事务以及查看状态等。

它并不会和RM产生联系。也就是说它不会负责去请求RM的第一阶段，这是和其他平台不同的一点。

TC全程接管和更新RM状态。

分支状态

![global](https://cdn.syst.top/state3.png)



项目地址:

Easycar: https://github.com/wuqinqiang/easycar

Client-go: https://github.com/easycar/client-go

Examples： https://github.com/easycar/examples

