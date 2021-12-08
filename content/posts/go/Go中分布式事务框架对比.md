---
title: "那些用Go实现的分布式事务框架"
date: 2021-12-13T22:25:52+08:00 
toc: true 
tags:

- go
- enum

---

### 开篇

不知不觉竟然一个月没更新了，人一旦懒下来只会越来越懒。

最近对分布式事务产生了一些兴趣，查阅了一些文章以及论文。这篇文章主要介绍我看的两个项目，不涉及一些理论知识。

- 阿里开源版本的Seata，主要看了Go实现的seata-golang(落后java版)
- 以及前段时间很多公众号都发的dtm。



### Seata简介

Seata是由阿里开源的分布式事务服务，目前为用户提供了AT、TCC、SAGA、XA的事务模式，整体采用的是两阶段提交协议。Go版的seata-golang 目前好像只实现了mysql的AT、TCC模式，作者现在不咋更新了。



Seata 有几个核心角色：

- TC(Transaction Coordinator) -事务协调者。(维护全局和分支事务的状态，驱动全局事务提交或回滚)
- TM(Transaction Manager)-事务管理器。(定义全局事务的范围：开始全局事务、提交或回滚全局事务。)
- RM(Resource Manager)-资源管理器。(管理分支事务处理的资源，与TC交谈以注册分支事务和报告分支事务的状态，并驱动分支事务提交或回滚)



当然这样看，可能还不是很理解，我拿一张官网的图加以解释。

![seata_tcc-1](https://cdn.syst.top/transaction.png)

从上图中可以看出，这三个角色所负责的工作如下，

TC

- 维护全局和分支事务状态，需要进行存储。
- 当一个分布式事务处理结束，需要通知到每个RM是commit还是rollback。

TM 

- 向TC请求开启一个分布式事务，得到一个全局唯一的分布式id。
- 根据每个参与分布式事务的RM一阶段的反馈，决定二阶段向TC请求此次分布式事务是commit还是rollback(绝大部分场景下，一阶段任一RM失败，本次分布式事务失败)

RM

说的白一点就是管理参与分布式事务的各个服务(比如经典下单场景中涉及到的:订单服务、库存服务、营销服务等)

ps:个人感觉，这里的RM有点类似微服务中的中间处理层(专业术语他们管这叫bff->backend for fronted)。

- 一阶段 prepare 行为(主动)：每个RM调用 **自定义** 的 prepare 逻辑。

- 二阶段 commit 行为(被动触发)：如果本次分布式事务第一阶段全部RM成功，TC处理完自身状态变更后，调用各个RM**自定义** 的 commit 逻辑。(一阶段RM全部成功)
- 二阶段 rollback 行为(被动触发)：如果本次分布式事务第一阶段任一RM失败，TC处理完自身状态变更后，调用各个RM**自定义** 的 rollback 逻辑。(一阶段任意RM失败)



好了。下面可以看看seata-golang 实现的一些细节了，seata-golang 底层采用gRPC进行通信。

### seata-golang

我们先看RM部分结构。

```go
type ResourceManager struct {
	addressing     string //rm地址
	rpcClient      apis.ResourceManagerServiceClient //rm rpc客户端
  managers       map[apis.BranchSession_BranchType]ResourceManagerInterface //存储rm事务模式（比如TCC、AT等）
	branchMessages chan *apis.BranchMessage //存储将要向TC响应的消息
}
```

```go
type ResourceManagerServiceClient interface {
  // 和 TC 流数据交互接口
   BranchCommunicate(ctx context.Context, opts ...grpc.CallOption) (ResourceManagerService_BranchCommunicateClient, error)
  // 向TC 注册分支
   BranchRegister(ctx context.Context, in *BranchRegisterRequest, opts ...grpc.CallOption) (*BranchRegisterResponse, error)
  //.....
}
```

至于managers，保存支持的各大事务模式(TCC等)，每个模式只需要实现此接口即可。

```go
type ResourceManagerInterface interface {
  // 分支提交
	BranchCommit(ctx context.Context, request *apis.BranchCommitRequest) (*apis.BranchCommitResponse, error)
 // 分支回退
	BranchRollback(ctx context.Context, request *apis.BranchRollbackRequest) (*apis.BranchRollbackResponse, error)
  // ......
}
```

再看TC部分结构(只保留会涉及的字段，其他细节可以自行查阅)。

```go
type TransactionCoordinator struct {
  // ....
	holder             *holder.SessionHolder //数据存储
  // ......
  futures            *sync.Map //临时存储RM响应数据，有g在接受处理
  // ......
	callBackMessages   *sync.Map //临时存储TC即将向RM发送的消息,有格外g在发送数据
}


type SessionHolder struct {
	manager storage.SessionManager
}


// SessionManager stored the globalTransactions and branchTransactions.
type SessionManager interface {
	// Add global session.
	AddGlobalSession(session *apis.GlobalSession) error

	// Find global session.
	FindGlobalSession(xid string) *apis.GlobalSession

	// Find global sessions list.
	FindGlobalSessions(statuses []apis.GlobalSession_GlobalStatus) []*apis.GlobalSession
}



```

TC对数据的存储目前支持mysql和pgsql，即实现SessionManager接口，然后注入到SessionHolder的manager。



介绍完这两个基本结构，还记得我们上面说过他们之间的关系吗？



二阶段TC会根据当前事务状态去通知RM是commit还是rollback。

在初始化ResourceManager 的时候，

```go
func InitResourceManager(addressing string, client apis.ResourceManagerServiceClient) {
	defaultResourceManager = &ResourceManager{
		addressing:     addressing,
		rpcClient:      client,
		managers:       make(map[apis.BranchSession_BranchType]ResourceManagerInterface),
		branchMessages: make(chan *apis.BranchMessage),
	}
	runtime.GoWithRecover(func() {
    // 会单独开启一个g调用
		defaultResourceManager.branchCommunicate()
	}, nil)
}
```

```go
func (manager *ResourceManager) branchCommunicate() {
     //省略代码。。。。。。。
      stream, err := manager.rpcClient.BranchCommunicate(ctx)
      // 省略代码
}
```

我们看到最终会调用TC一个 grpc 接口branchCommunicate。

```go
	BranchCommunicate(ctx context.Context, opts ...grpc.CallOption) (ResourceManagerService_BranchCommunicateClient, error)
```

```go
type ResourceManagerService_BranchCommunicateClient interface {
	Send(*BranchMessage) error
	Recv() (*BranchMessage, error)
	grpc.ClientStream
}
```

对应到服务端。

```go
rpc BranchCommunicate(stream BranchMessage) returns (stream BranchMessage);
```

```go
type ResourceManagerService_BranchCommunicateServer interface {
	Send(*BranchMessage) error
	Recv() (*BranchMessage, error)
	grpc.ServerStream
}
```

我们知道gRPC有四种基础的通信模式。

- 一元模式（Unary RPC）
- 服务器端流RPC（Server Sreaming RPC）
- 客户端流RPC（Client Streaming RPC）
- 双向流RPC（Bidirectional Streaming RPC）

想要流的形式也很简单，只需要在proto方法定义中将对应的请求|响应 参数前加上stream标记，那么这个接口就是流式传送了。至于是哪种流，取决于你把stream加在哪边，如果请求和响应都加，那么就是双向流了。

客户端和服务端都可以通过stream.Send 发送请求，通过stream.Recv 读取响应。

当RM调用BranchCommunicate时，

```go
func (manager *ResourceManager) branchCommunicate() {
   for {
      ctx := metadata.AppendToOutgoingContext(context.Background(), "addressing", manager.addressing)
      // 得到steam
      stream, err := manager.rpcClient.BranchCommunicate(ctx)
      // 即->
//      type ResourceManagerService_BranchCommunicateClient interface {
//       	Send(*BranchMessage) error
//	      Recv() (*BranchMessage, error)
//      	grpc.ClientStream
//       }

      if err != nil {
         time.Sleep(time.Second)
         continue
      }

      done := make(chan bool)
      // 另开一个g用来响应消息给TC
      runtime.GoWithRecover(func() {
      // 死循环
         for {
            select {
            case _, ok := <-done:
               if !ok {
                  return
               }
               // 如果branchMessages 有数据，说明是处理完tc的响应数据，通过send响应给tc
            case msg := <-manager.branchMessages:
               err := stream.Send(msg)
               if err != nil {
                  return
               }
            }
         }
      }, nil)


// 另一个死循环是用来接受从tc发送过来的数据
      for {
      // 接受数据
         msg, err := stream.Recv()
         if err == io.EOF {
            close(done)
            break
         }
         if err != nil {
            close(done)
            break
         }
         // 判断tc发送数据的类型，是提交分支事务类型还是回滚类型
         switch msg.BranchMessageType {
         // 如果是分支提交的消息
         case apis.TypeBranchCommit:
            request := &apis.BranchCommitRequest{}
            data := msg.GetMessage().GetValue()
            err := request.Unmarshal(data)
            if err != nil {
               log.Error(err)
               continue
            }
            // 处理分支提交，得到处理结果
            response, err := manager.BranchCommit(context.Background(), request)
            if err == nil {
               content, err := types.MarshalAny(response)
               if err == nil {
               // 再把处理完的结果包装一下塞到branchMessages等待发送出去
                  manager.branchMessages <- &apis.BranchMessage{
                     ID:                msg.ID,
                     BranchMessageType: apis.TypeBranchCommitResult,
                     Message:           content,
                  }
               }
            }
            
            // 如果是回滚的消息，本质上流程差不多
         case apis.TypeBranchRollback:
            request := &apis.BranchRollbackRequest{}
            data := msg.GetMessage().GetValue()
            err := request.Unmarshal(data)
            if err != nil {
               log.Error(err)
               continue
            }
            response, err := manager.BranchRollback(context.Background(), request)
            if err == nil {
               content, err := types.MarshalAny(response)
               if err == nil {
                  manager.branchMessages <- &apis.BranchMessage{
                     ID:                msg.ID,
                     BranchMessageType: apis.TypeBranchRollBackResult,
                     Message:           content,
                  }
               }
            }
         }
      }
      // 关闭流
      err = stream.CloseSend()
      if err != nil {
         log.Error(err)
      }
   }
}
```

最终处理分支事务调用manager.BranchCommit，

```go
func (manager ResourceManager) BranchCommit(ctx context.Context, request *apis.BranchCommitRequest) (*apis.BranchCommitResponse, error) {
// 根据事务模式调用对应的处理
	rm, ok := manager.managers[request.BranchType]
	if ok {
		return rm.BranchCommit(ctx, request)
	}
	return &apis.BranchCommitResponse{
		ResultCode: apis.ResultCodeFailed,
		Message:    fmt.Sprintf("there is no resource manager for %s", request.BranchType.String()),
	}, nil
}
```

相应的，当TC被RM调用BranchCommunicate 后，

```go
func (tc *TransactionCoordinator) BranchCommunicate(stream apis.ResourceManagerService_BranchCommunicateServer) error {
	var addressing string
	done := make(chan bool)

	ctx := stream.Context()
	md, ok := metadata.FromIncomingContext(ctx)
  
  //省略一些无关代码
  //............
	addressing = md.Get("addressing")[0]

 //这里很好懂吧
	queue, _ := tc.callBackMessages.LoadOrStore(addressing, make(chan *apis.BranchMessage))
	q := queue.(chan *apis.BranchMessage)

  
  // 单独一个g 运行
  // 用来从callBackMessages中取数据发送给客户端
	runtime.GoWithRecover(func() {
    // 死循环
		for {
			select {
			case _, ok := <-done:
				if !ok {
					return
				}
        // 如果能拿到数据，说明有数据需要发送给客户端
			case msg := <- q:
				err := stream.Send(msg)
				if err != nil {
					return
				}
			}
		}
	}, nil)

  // 下面的死循环逻辑主要是从stearm 接受客户端响应数据，然后塞入到future中，根据唯一的branch_message_Id
	for {
		select {
		case <-ctx.Done():
			close(done)
			return ctx.Err()
		default:
      //有数据
			branchMessage, err := stream.Recv()
			if err == io.EOF {
				close(done)
				return nil
			}
			if err != nil {
				close(done)
				return err
			}
      // 要区分是什么响应
      // 根据是 事务commit的响应，还是对rollback的响应,交给不同逻辑处理。
			switch branchMessage.GetBranchMessageType() {
        // 分支事务提交的响应
			case apis.TypeBranchCommitResult:
				response := &apis.BranchCommitResponse{}
				data := branchMessage.GetMessage().GetValue()
        // 解析数据
				err := response.Unmarshal(data)
				if err != nil {
					log.Error(err)
					continue
				}
        
        // 说明发送commit的动作在等待响应
				resp, loaded := tc.futures.Load(branchMessage.ID)
				if loaded {
          // 赋值响应数据
					future := resp.(*common2.MessageFuture)
					future.Response = response
          // 通知数据到了
					future.Done <- true
					tc.futures.Delete(branchMessage.ID)
				}
        // 和上面差不多逻辑除了响应数据不同
			case apis.TypeBranchRollBackResult:
				response := &apis.BranchRollbackResponse{}
				data := branchMessage.GetMessage().GetValue()
				err := response.Unmarshal(data)
				if err != nil {
					log.Error(err)
					continue
				}
				resp, loaded := tc.futures.Load(branchMessage.ID)
				if loaded {
					future := resp.(*common2.MessageFuture)
					future.Response = response
					future.Done <- true
					tc.futures.Delete(branchMessage.ID)
				}
			}
		}
	}
}
```

上面要发送给RM 通知commit 或者 rollback 数据是咋么来的呢？

当TC要通知RM进行分支commit 的时候，

```go
func (tc *TransactionCoordinator) branchCommit(bs *apis.BranchSession) (apis.BranchSession_BranchStatus, error) {
   request := &apis.BranchCommitRequest{
      XID:             bs.XID,
      BranchID:        bs.BranchID,
      ResourceID:      bs.ResourceID,
      LockKey:         bs.LockKey,
      BranchType:      bs.Type,
      ApplicationData: bs.ApplicationData,
   }

   content, err := types.MarshalAny(request)
   if err != nil {
      return bs.Status, err
   }
   message := &apis.BranchMessage{
      ID:                int64(tc.idGenerator.Inc()),
      BranchMessageType: apis.TypeBranchCommit,
      Message:           content,
   }
  
    //上面是封装数据
   queue, _ := tc.callBackMessages.LoadOrStore(bs.Addressing, make(chan *apis.BranchMessage))
   q := queue.(chan *apis.BranchMessage)
   select {
     // 在这把数据塞到callBackMessages
   case q <- message:
   default:
      return bs.Status, err
   }

  // 这里创建了RM响应格式，根据 messageid，塞入到结果futures(sync.map中)，
   resp := common2.NewMessageFuture(message)
   tc.futures.Store(message.ID, resp)

   timer := time.NewTimer(tc.streamMessageTimeout)
   select {
     // RM响应超时了，GG
   case <-timer.C:
      tc.futures.Delete(resp.ID)
      return bs.Status, fmt.Errorf("wait branch commit response timeout")
   case <-resp.Done:
      timer.Stop()
   }

  // 响应成功，解析数据处理。
   response, ok := resp.Response.(*apis.BranchCommitResponse)
   if !ok {
      log.Infof("rollback response: %v", resp.Response)
      return bs.Status, fmt.Errorf("response type not right")
   }
   if response.ResultCode == apis.ResultCodeSuccess {
      return response.BranchStatus, nil
   }
   return bs.Status, fmt.Errorf(response.Message)
}
```

最后一个就是TM，没啥理解难度。

```go
// tm request to  tc
type TransactionManagerInterface interface {
	// GlobalStatus_Begin a new global transaction.
	Begin(ctx context.Context, name string, timeout int32) (string, error)

	// Global commit.
	Commit(ctx context.Context, xid string) (apis.GlobalSession_GlobalStatus, error)

	// Global rollback.
	Rollback(ctx context.Context, xid string) (apis.GlobalSession_GlobalStatus, error)

	// Get current status of the give transaction.
	GetStatus(ctx context.Context, xid string) (apis.GlobalSession_GlobalStatus, error)

	// Global report.
	GlobalReport(ctx context.Context, xid string, globalStatus apis.GlobalSession_GlobalStatus) (apis.GlobalSession_GlobalStatus, error)
}

type TransactionManager struct {
	addressing string
	rpcClient  apis.TransactionManagerServiceClient
}

func InitTransactionManager(addressing string, client apis.TransactionManagerServiceClient) {
	defaultTransactionManager = &TransactionManager{
		addressing: addressing,
		rpcClient:  client,
	}
}
```

其实seat-golang还有别的可以提一提的。



比如说，它里面通过go反射实现的动态代理功能(虽然我觉得完全没必要?)，我懒得写了。



这篇文章再写就更长了，不继续写dtm了，感兴趣的留个言，我看看要不要写一篇dtm。



