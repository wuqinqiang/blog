---
title: "Gnet原理解析"
date: 2022-06-07T16:23:51+08:00 
toc: true 
tags :

- go
- net

---

距离上次写文章过了一月有余，这段时间着实太躺了。以至于昨晚做了一个噩梦，醒来的时候狠狠的抽了自己两巴掌，不能这么躺了。

上面当然是个笑话。



### 开篇

上一篇我们分析了Go原生网络模型以及部分源码，绝大部分场景下(99%)，使用原生netpoll已经足够了。

但是在一些海量并发连接下，原生netpoll会为每一个连接都开启一个goroutine处理，也就是1千万的连接就会创建一千万个goroutine。这就给了这些特殊场景下的优化空间，这也是像gnet和cloudwego/netpoll诞生的原因之一吧。

本质上他们的底层核心都是一样的，都是基于epoll(linux)实现的。

只是对事件发生后，每个库的处理方式会有所不同。

本篇文章主要分析gnet的。至于使用姿势就不发了，gnet有对应的demo库，可以自行体验。



### 架构

直接引用gnet官网的一张图

![截屏2022-06-02 15.03.18](https://cdn.syst.top/net1.png)

gnet采用的是『主从多 Reactors』。也就是一个主线程负责监听端口连接，当一个客户端连接到来时，就把这个连接根据负载均衡算法分配给其中一个sub线程，由对应的sub线程去处理这个连接的读写事件以及管理它的死亡。



下面这张图就更清晰了。

![图片来源gnet官网](https://cdn.syst.top/net2.png)



### 核心结构

我们先解释gnet的一些核心结构。

![image-20220602153051260](https://cdn.syst.top/net4.png)

engine就是程序最上层的结构了。

- ln对应的listener就是服务启动后对应监听端口的监听器。

- lb对应的loadBalancer就是负载均衡器。也就是当客户端连接服务时，负载均衡器会选择一个sub线程，把连接交给此线程处理。

- mainLoop 就是我们的主线程了，对应的结构eventloop。当然我们的sub线程结构也是eventloop。结构相同，不同的是职责。主线程负责的是监听端口发生的客户端连接事件，然后再由负载均衡器把连接分配给一个sub线程。而sub线程负责的是绑定分配给他的连接(不止一个)，且等待自己管理的所有连接后续读写事件，并进行处理。



接着看eventloop。

![image-20220602154501439](https://cdn.syst.top/net5.png)

- netpoll.Poller:每一个 eventloop都对应一个epoll或者kqueue。
- buffer用来作为读消息的缓冲区。
- connCoun记录当前eventloop存储的tcp连接数。
-  udpSockets和connetcions分别管理着这个eventloop下所有的udp socket和tcp连接，注意他们的结构map。这里的int类型存储的就是fd。



对应conn结构，

![image-20220608175339906](https://cdn.syst.top/net5-1.png)

这里面有几个字段介绍下，

- buffer:存储当前conn对端(client)发送的最新数据，比如发送了三次，那个此时buffer存储的是第三次的数据,代码里有。
- inboundBuffer:存储对端发送的且未被用户读取的剩余数据，还是个Ring Buffer。
- outboundBuffer:存储还未发送给对端的数据。(比如服务端响应客户端的数据，由于conn fd是不阻塞的，调用write返回不可写的时候，就可以先把数据放到这里)

conn相当于每个连接都会有自己独立的缓存空间。这样做是为了减少集中式管理内存带来的锁问题。使用Ring buffer是为了增加空间的复用性。

整体结构就这些。



### 核心逻辑

当程序启动时，

![net3](https://cdn.syst.top/net3.png)

会根据用户设置的options明确eventloop循环的数量，也就是有多少个sub线程。再进一步说，在linux环境就是会创建多少个epoll对象。

那么整个程序的epoll对象就是count(sub)+1(main Listener)。

![image-20220608164415565](https://cdn.syst.top/net6.png)

上图就是我说的，会根据设置的数量创建对应的eventloop,把对应的eventloop 注册到负载均衡器中。

当新连接到来时，就可以根据一定的算法(gnet提供了轮询、最少连接以及hash)挑选其中一个eventloop把连接分配给它。



我们先来看主线程，(由于我使用的是mac,所以后面关于IO多路复用，实现部分就是kqueue代码了，当然原理是一样的)

![image-20220608165350443](https://cdn.syst.top/net7.png)

Polling就是等待网络事件到来，传递了一个闭包参数，更确切的说是一个事件到来时的回调函数，从名字可以看出，就是处理新连接的。

至于Polling函数，

![image-20220608170912204](https://cdn.syst.top/net8.png)

逻辑很简单，一个for循环等待事件到来，然后处理事件。

主线程的事件分两种，

一种是正常的fd发生网络连接事件，

一种是通过NOTE_TRIGGER立即激活的事件。

![image-20220608171537730](https://cdn.syst.top/net9.png)

通过NOTE_TRIGGER触发告诉你队列里有task任务，去执行task任务。

如果是正常的网络事件到来，就处理闭包函数，主线程处理的就是上面的accept连接函数。

![image-20220608172333749](https://cdn.syst.top/net10.png)

accept连接逻辑很简单，拿到连接的fd。设置fd非阻塞模式(想想连接是阻塞的会咋么样?),然后根据负载均衡算法选择一个sub 线程，通过register函数把此连接分配给它。

![image-20220608173157607](https://cdn.syst.top/net11.png)

register做了两件事，首先需要把当前连接注册到当前sub 线程的epoll or kqueue 对象中,新增read的flag。

接着就是把当前连接放入到connections的map结构中 fd->conn。

这样当对应的sub线程事件到来时，可以通过事件的fd找到是哪个连接，进行相应的处理。

![image-20220608174647390](https://cdn.syst.top/net12.png)

 如果是可读事件，

![image-20220608182813931](https://cdn.syst.top/net13.png)

到这里分析差不多就结束了。



### 总结

在gnet里面，你可以看到，基本上所有的操作都无锁的。

那是因为事件到来时，采取的都是非阻塞的操作，且是串行处理对应的每个fd(conn)。每个conn操作的都是自身持有的缓存空间。同时处理完一轮触发的所有事件才会循环进入下一次等待，在此层面上解决了并发问题。

当然这样用户在使用的时候也需要注意一些问题，比如用户在自定义EventHandler中，如果要异步处理逻辑，就不能像下面这样开一个g然后在里面获取本次数据，

![image-20220608184718192](https://cdn.syst.top/net14.png)

而应该先拿到数据，再异步处理。

![image-20220608184937855](https://cdn.syst.top/net15.png)

issues上有提到，连接是使用map[int]*conn存储的。gnet本身的场景就是海量并发连接，内存会很大。进而big map存指针会对 GC造成很大的负担，毕竟它不像数组一样，是连续内存空间，易于GC扫描。

还有一点，在处理buffer数据的时候，就像上面看到的，本质上是将buffer数据copy给用户一份，那么就存在大量copy开销,在这一点上，字节的netpoll实现了Nocopy Buffer，改天研究一下。
