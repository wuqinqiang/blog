---
title: "超实用的 gRPC 客户端调试工具"
date: 2021-07-27T10:01:52+08:00 
toc: true 
tags:

- grpc
- go

---

### 介绍

正好看到董泽润老哥写了一篇关于使用 `WireShark` 分析 `gRPC` 流量的文章，学到了。

那我就介绍两个日常开发使用过的两款 `gRPC` 客户端调试工具吧。

## Evans

`Evans` 有两种模式：`REPL` 和 `CLI`。比起其他 `gRPC` 客户端,更具有表现力。并且它还支持自动补全功能。

它的安装非常方便，在 `Mac` 上我们只需要执行以下两行命令即可。

```go
$ brew tap ktr0731/evans
$ brew install evans
```
我们来操作一下 `REPL` 模式。

首先我们需要有一个 `pb` 文件，假设你的文件在 `api/api.proto`，我们只需要这样：
![image](https://image.syst.top/image/grpc/1.gif)

默认地址为 `127.0.0.1:50051`，当然你可以通过 `--host` 和 `--port` 来指定服务器。
![image](https://image.syst.top/image/grpc/2.png)

上图的命令:
 - `show package` 读取 `pb` 包名，
- `show service` 显示对应服务列表。
-  `call xxx` 调用 `grpc` 服务......
- .....

更多命令可自行查阅官网。

除了上述这种直接引入 `pb` 文件外，我们还可以通过 `gRPC` 反射包(`reflection`)， 将 `grpc.Server` 注册到反射服务中,
这样的话，就可以通过 `reflection` 提供的反射服务查询到对应的 `gRPC` 服务或者调用 `gRPC` 服务。 

操作起来很简单，

```go
import (
"google.golang.org/grpc"
"google.golang.org/grpc/reflection"
)
	grpcServer := grpc.NewServer()
	reflection.Register(grpcServer)
```

回到 `Evans` 工具， 如果一个 `gRPC` 服务启动了反射，我们就可以使用 `-r (--reflection)` 选项来启动 `Evans`。

比如像下面这样：
![image](https://image.syst.top/image/grpc/3.gif)


### BloomRPC

`BloomRPC` 是一个简单的`GUI` 客户端工具，使用这个那就更简单了。

只需要导入 `pb` 文件，然后点两下即可。
![image](https://image.syst.top/image/grpc/4.gif)

当然有个不好点在于，每次修改了 `pb`，都不得不重新导入。


### 总结
以上介绍了两款 `gRPC` 客户端工具。不知道你们平常都使用 `gRPC` 哪些周边工具，欢迎一起讨论。当然调试工具再好，改写的 `test` 还是逃不了。

