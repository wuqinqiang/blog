---
title: "socks5结合抓包详解"
date: 2023-06-29T15:20:00+08:00 
toc: true 
tags :

- go
- net


---

SOCKS5（Socket Secure 5）是一种网络协议，用于在客户端和代理服务器之间进行通信。它是SOCKS协议的第五个版本，SOCKS5协议支持TCP和UDP协议，并提供了认证和加密的功能。

SOCKS5协议广泛用于代理服务器、xx上网、匿名访问、负载均衡等场景。它提供了一种通用的、灵活的代理解决方案，可以在各种网络环境和应用中使用。

RFC 1928是关于SOCKS5的协议文档。



**协议解析**

SOCKS5工作在应用层和传输层之间，首先客户端需要和代理服务器进行tcp连接。

连接完成后，客户端和代理服务器之间进行协商确定认证方式。

具体就是客户端发送认证方法请求，

![截屏2023-06-29 11.19.11](https://cdn.syst.top/rfc1928-1.png)



- VER字段占1个字节，表示SOCKS协议的版本号，目前为5（即SOCKS5）。
- NMETHODS字段占1个字节，表示客户端支持的认证方法数量。
- METHODS字段占N个字节，表示客户端支持的认证方法列表。每个字节代表一种认证方法，取值范围为1到255，对应不同的认证方式。



代理服务器在收到客户端消息，从客户端提供的METHODS字段中选择一种支持的认证方法，并将该方法的标识填充到METHOD字段中。代理服务器通过这个应答消息告知客户端选择的认证方法。

![截屏2023-06-29 11.34.41](https://cdn.syst.top/rfc1928-2.png)

目前的METHOD包含以下几个值：

- X'00'：NO AUTHENTICATION REQUIRED。表示客户端无需进行任何认证，可以直接进行连接或操作。
- X'01'：GSSAPI。表示使用GSSAPI（Generic Security Services Application Program Interface）进行认证。
- X'02'：USERNAME/PASSWORD。表示客户端需要使用用户名和密码进行认证。
- X'03'到X'7F'：由IANA（Internet Assigned Numbers Authority）分配的认证方法。这些方法可能具有特定的定义和用途，可以根据具体的分配情况来确定其含义。
- X'80'到X'FE'：保留给私有方法（RESERVED FOR PRIVATE METHODS）。这些方法可能由特定实现或组织自定义，不在通用的认证方法范围内。
- X'FF'：无可接受的方法（NO ACCEPTABLE METHODS）。表示代理服务器无法接受客户端提供的任何认证方法，无法进行连接或操作。

如果返回的方法是X 'FF'，意味着失败了，那么客户端需要关闭连接。

一旦协商完毕，客户端会发送请求的详细信息，主要包括实际要请求的目的地址和端口号。

![截屏2023-06-29 13.16.06](https://cdn.syst.top/rfc1928-3.png)

其中各字段含义如下：

- VER：协议版本，固定为X'05'。
- CMD：连接命令，指定客户端的操作类型。常见的取值有：
  - X'01'：CONNECT。
  - X'02'：BIND。
  - X'03'：UDP。
- RSV：保留字段，固定为X'00'。
- ATYP：目标地址类型，指定DST.ADDR字段的类型。常见的取值有：
  - X'01'：IPv4地址。
  - X'03'：域名。
  - X'04'：IPv6地址。
- DST.ADDR：目标地址，根据ATYP字段的类型来确定具体的格式。
- DST.PORT：目标端口，表示客户端要连接的目标服务器的端口号。

代理服务器收到请求后，会发起到DST.ADDR:PORT的连接，并响应客户端结果，

![截屏2023-06-29 13.43.05](https://cdn.syst.top/rfc1928-4.png)

其中的值有：

REP:

- X'00' 成功
- X'01' SOCKS服务器故障
- X'02' 连接不符合规则
- X'03' 网络不可达
- X'04' 主机不可达
- X'05' 连接被拒绝  
- X'06' TTL 过期 
- X'07' 命令不支持
- X'08' 地址类型不支持
- X'09' to X'FF' 未分配

RSV：预留位，必须设置成X'00'。

BND.ADDR: 服务器绑定的地址

BND.PORT: 服务器绑定的端口（网络字节序）

当连接建立后，客户端就可以和正常一样访问目标地址"通信"了。此时通信的数据除了目的地址是发往代理服务器以外，所有内容都是和普通连接一样。对代理服务器而言，后面所有收到的来自客户端的数据都会原样转发到目标服务。



**抓个包看看**

实验涉及到的两个库:

- https://github.com/shadowsocks/go-shadowsocks2
- https://github.com/haad/proxychains

假设我现在有两台服务器，服务器A(上海)和服务器B(伦敦),服务器B上部署了Shadowsocks服务，Shadowsocks 是一个安全的socks5代理。

我想把机器A对外请求的流量都通过B来代理，并且指定只有发送到A机器的1080端口流量才能被代理。

通过go-shadowsocks2启动程序，

```shell
./shadowsocks2 -c 'ss://AES-256-GCM:password@b机器ip:port' -socks 127.0.0.1:1080 
```

参数说明，

-c: 设置Shadowsocks2客户端连接到B机器Shadowsocks的配置。

-socks:  指定Shadowsocks2客户端在本地监听的SOCKS5代理服务器的地址。在这个例子中，客户端在1080端口上监听代理请求。

myip.ipip.net是一个用来查询本机公网ip的服务。假设我们直接在服务器A上curl myip.ipip.net，那么请求的结果，

![截屏2023-06-29 15.44.36](https://cdn.syst.top/rfc1928-5.png)

现在我们要通过上面描述的达到输出的是B服务器伦敦的ip地址。

我们使用了工具proxychains，配置proxychains，

![截屏2023-06-29 17.09.08](https://cdn.syst.top/rfc1928-6.png)



执行之前，我们先开启tcpdump，我们只抓本地1080端口的流量，毕竟它是作为本地其他客户端socks5代理服务器。

```shell
tcpdump -i lo port 1080 -w ss.pcap
```

然后执行，

```shell
proxychains4 curl myip.ipip.net
```

![截屏2023-06-29 15.45.12](https://cdn.syst.top/rfc1928-7.png)



最终获取抓到的包。

![截屏2023-06-29 17.35.59](https://cdn.syst.top/rfc1928-8.png)

前三个包是tcp的三次握手，这里就不说了。

包4也就是客户端发送协商认证方法请求给代理服务器，其中包含支持的认证方法列表，这里只支持1种方法:0。也就是客户端无需进行任何认证，可以直接进行连接或操作。

![截屏2023-06-29 17.41.35](https://cdn.syst.top/rfc1928-9.png)

包6服务端响应，

![截屏2023-06-29 17.41.58](https://cdn.syst.top/rfc1928-10.png)

此时协商完毕。

包8是客户端发起请求的详细信息，里面包含了要访问的目标地址和端口。这里的address type=3 表示域名类型。

![截屏2023-06-29 17.46.11](https://cdn.syst.top/rfc1928-11.png)

包9为服务端响应:

![截屏2023-06-29 17.47.51](https://cdn.syst.top/rfc1928-12.png)

可以看到 RSV的值表示成功。

这里要注意的是包8中的CMD参数。上面说到有三个值。当客户端发往代理服务器的数据包的 CMD 字段的值为 0x01 时, 代表 CONNECT, 此时 DST.ADDR 和 DST.PORT 表示客户端所想要访问的目标主机的地址和端口, 代理服务器在收到该请求后建立代理服务器到目标主机的 TCP 连接, 并将代理服务器分配的 IP 地址和端口在返回的数据包中的 BIND.ADDR 和 BIND.PORT 表示。其他CMD值含义自行查看rfc。

但是包9中在对CONNECT请求回复中，BIND.PORT 和BIND.ADDR都不是预期的值。看了shadowsocks2里面的代码，

![截屏2023-06-29 18.04.55](https://cdn.syst.top/rfc1928-13.png)

它是直接把ATYP设置成1，BND.ADDR和BND.PORT设置为0。

因为此时的本地socks5并不是最终的代理服务器，实际请求目标地址的是我们在启动shadowsocks2配置的B机器上的ss服务。本身BND.ADDR和BND.PORT字段是可选的，如果服务器不提供这些信息，客户端可以根据实际情况进行处理。

当响应成功，客户端开始请求传输数据了，也就是包10，发送了一个http请求。包12是对请求的响应。

![截屏2023-06-29 20.49.48](https://cdn.syst.top/rfc1928-14.png)

所以实际的流转是,

local A client <-socks5-> A:1080 <-> b机器Shadowsocks <-> myip.ipip.net

在Shadowsocks2中，主要通过这个函数来实现双向数据传输:

```go
// relay copies between left and right bidirectionally
func relay(left, right net.Conn) error {
	var err, err1 error
	var wg sync.WaitGroup
	var wait = 5 * time.Second
	wg.Add(1)
	go func() {
		defer wg.Done()
		_, err1 = io.Copy(right, left)
		right.SetReadDeadline(time.Now().Add(wait)) // unblock read on right
	}()
	_, err = io.Copy(left, right)
	left.SetReadDeadline(time.Now().Add(wait)) // unblock read on left
	wg.Wait()
	if err1 != nil && !errors.Is(err1, os.ErrDeadlineExceeded) { // requires Go 1.15+
		return err1
	}
	if err != nil && !errors.Is(err, os.ErrDeadlineExceeded) {
		return err
	}
	return nil
}

```

也就是，(client <-socks5-> A:1080 )  <->  (Shadowsocks <-> myip.ipip.net)

主要靠的是标准库的io.Copy。

最后13-16包就是tcp的4次挥手了。

上图这组包还有一个很基础的tcp知识点，

![截屏2023-06-29 17.35.59](https://cdn.syst.top/rfc1928-8.png)

首先你图上看到的seq都是从0开始的，那是因为wireshark有个可以开启Relative Sequence Numbers的选项。

上面的包4截图可以看到包信息:Seq =1,Len=3。再看包5，是对包4的Ack, 包6包7同理。

但是你再看包8：Seq:4，Len=20。

![截屏2023-06-29 21.16.14](https://cdn.syst.top/rfc1928-15.png)



但是你在包9看到的也是一个Socks协议的包，并没有看到专门对包8的Ack。

在TCP通信中，并不一定每个请求都会立即收到相应的ACK确认包。TCP使用了一种称为"累积确认"（Cumulative Acknowledgment）的机制。ACK确认包可能会延迟发送，或者包含在接收方发送的其他数据包中。这是TCP协议的一种优化机制，可以减少网络中的包数量和传输开销。

那么我们就知道对包8的确认，其实是随着包9的数据一起发送了。Ack=Seq(4)+Len(20)=24

![截屏2023-06-29 21.21.37](https://cdn.syst.top/rfc1928-16.png)

完结。
