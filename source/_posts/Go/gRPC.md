---
title: Something in gRPC 
date: 2019-06-29 14:40
tags: golang
---

常用的rpc框架

<!--more-->
## 整体模型

就是常见的一种`Reactor`模型，

listen线程和处理线程是不同的
## 连接池



## 服务发现和服务注册

很遗憾，grpc只提供了接口,这些是要自己用其他服务发现的中间件来实现

### 一般的设计

#### 集中式LB

就是在server和client端中间加一个proxy来记录注册的服务（一些地址映射表）；

- 单点问题(proxy改成分布式组件即可)
- 增加了额外中间件，复杂度提高

#### 进程内LB(Balancing-aware client)

因为第一个问题，那么我们把这个proxy放到消费方进程里，这个可以叫软负载或者客户端负载；
定期用心跳来同步服务注册表来表明服务存活状态

- LB和服务发现分散到每一个服务消费者进程内部
- 同时，消费方和提供方是直接连接，无消耗

- 开发成本高，不同语言的调用方要有不同语言版本的LB
- 升级后，每个都要重新发布

#### 独立进程LB

同进程内LB差不多，都是在消费方进程中，只不过这个进程是独立出来专门负责服务发现和LB的

- 因为不同进程，简化服务调用方，不需要重新开发不同语言客户端
- 升级不用服务调用方改代码

### grpc的负载均衡设计

grpc就属于这种:
有通过第三方proxy获得负载均衡列表的方式(`grpclb`)，也有通过`Name Resolver`(dns)的方式(都是旧版本):
#### 新版本中(1.26以上)
主要通过 `atrributes.Attributes`来实现:

```go
type Address struct {
	...
	// Attributes contains arbitrary data about this address intended for
	// consumption by the load balancing policy.
	Attributes *attributes.Attributes

}

```
Atrributes包中我们可以看到,其为一个map，存储:
```go
// Attributes is an immutable struct for storing and retrieving generic
// key/value pairs.  Keys must be hashable, and users should define their own
// types for keys.
type Attributes struct {
	m map[interface{}]interface{}
}

```



#### grpclb
主要属于在客户端处实行负载均衡:
- 首先client会发起服务器名称解析请求，即把服务器名称解析成若干个IP，这些IP可能是负载均衡器的地址，也可能是实际服务器地址，同时可以设置是使用`grpclb`还是直接用roundRobin,roundRobinWeighted

- 然后就要实例化负载均衡策略,如果client知道某个ip是负载均衡地址，则client会使用`grpclb`策略，其他的设置为配置中要求的策略

- 负载均衡策略为每个服务建立一个subChannel（除了grpclb)
	这里要注意，grpclb下，会在`resolver`返回的负载均衡器IP上打开一个**流连接**，客户端会通过这个连接，根据名称获得需要的服务器IP
	但是，如果是非负载均衡器IP的话，会以回调的方式进行，以免没有均衡器

- 当有rpc请求时，就用负载均衡决定的Subchannel接收请求，当可用服务器为空时会被阻塞;

在注释中有一个图描述的比较清楚
```go
// The parent ClientConn should re-resolve when grpclb loses connection to the
// remote balancer. When the ClientConn inside grpclb gets a TransientFailure,
// it calls lbManualResolver.ResolveNow(), which calls parent ClientConn's
// ResolveNow, and eventually results in re-resolve happening in parent
// ClientConn's resolver (DNS for example).
//
//                          parent
//                          ClientConn
//  +-----------------------------------------------------------------+
//  |             parent          +---------------------------------+ |
//  | DNS         ClientConn      |  grpclb                         | |
//  | resolver    balancerWrapper |                                 | |
//  | +              +            |    grpclb          grpclb       | |
//  | |              |            |    ManualResolver  ClientConn   | |
//  | |              |            |     +              +            | |
//  | |              |            |     |              | Transient  | |
//  | |              |            |     |              | Failure    | |
//  | |              |            |     |  <---------  |            | |
//  | |              | <--------------- |  ResolveNow  |            | |
//  | |  <---------  | ResolveNow |     |              |            | |
//  | |  ResolveNow  |            |     |              |            | |
//  | |              |            |     |              |            | |
//  | +              +            |     +              +            | |
//  |                             +---------------------------------+ |
//  +-----------------------------------------------------------------+
// lbManualResolver is used by the ClientConn inside grpclb. It's a manual
// resolver with a special ResolveNow() function.
//
// When ResolveNow() is called, it calls ResolveNow() on the parent ClientConn,
// so when grpclb client lose contact with remote balancers, the parent
// ClientConn's resolver will re-resolve.
```
#### Name Resolver



### grpc服务发现

在`/resolver/`文件夹下(在`/naming/`文件夹下实现的接口已经报废):


```go

```
## http2

见我自己的http2的文章，随便记了一点;

### server端针对不同的帧进行处理

每个Stream有**唯一**的ID标识，如果是客户端创建的则ID是`奇数`，服务端创建的ID则是`偶数`。
如果一条连接上的ID使用完了，Client会新建一条连接，Server也会给Client发送一个`GOAWAY Frame`强制让Client新建一条连接;

一条grpc连接允许并发的发送和接收多个Stream，而控制的参数便是`MaxConcurrentStreams`，Golang的服务端默认是100。


### 超时问题

可以带上一个`timeout Context`，但是里面实际会转换成header frame中的 `grpc-timeout` ???

在grpc中，`header frame`会带上不少信息，比如`grpc-status`状态等等,
server端的处理可以看到`processHeaderField`方法中解析的多种header：
比较重要的有:
- `grpc-timeout`:长连接超时控制
- 

```go
func (d *decodeState) processHeaderField(f hpack.HeaderField) {
	switch f.Name {
		...
		case "grpc-timeout":
			d.data.timeoutSet = true
			var err error
			if d.data.timeout, err = decodeTimeout(f.Value); err != nil {
				d.data.grpcErr = status.Errorf(codes.Internal, "transport: malformed time-out: %v", err)
			}
		case ":path":
			d.data.method = f.Value
		case ":status":
			code, err := strconv.Atoi(f.Value)
			if err != nil {
				d.data.httpErr = status.Errorf(codes.Internal, "transport: malformed http-status: %v", err)
				return
			}
			d.data.httpStatus = &code
		case ....
	}
}
```

## 如何识别服务以及解析

### 识别服务
识别服务方面比较直接，就是用`string`判等服务名字即可;

解析数据主要是借用了`proto`的`generate`工具，生成了服务端和客户端的代码，里面使用了自带的解码器，跟pb文件结合起来，**避免了语言层面上的反射的消耗**;

我们这里只讨论go语法，语言方面其他都是大同小异
创建proto文件 **data.proto**

```go
syntax = "proto2";

service Authenticate{
	rpc login(toServerData) returns (ResponseFromServer){}
	rpc home(toServerData) returns(ResponseFromServer){}
	rpc logout(toServerData) returns(ResponseFromServer){}
}

message toServerData{
	required int32 ctype = 1;
	required string name =2;
    optional bytes httpdata=3;
}
message ResponseFromServer {
	required bool Success=1;
	optional bytes tcpData=2;
	// Errcode int
}
```

用安装的protoc插件生成
```s
protoc --proto_path=/mypath --go_out=plugins=grpc:. *.proto //当前在mypath路径下，用grpc模式生成proto文件
```
### proto解析

生成 **data.pb.proto**

-----------这些就直接参照手册理解吧-----------------  
这里还是记录下这厮的解码方式吧，参考
>> 《数据密集型系统设计》

protobuf的的编码其实跟 thrift的BinaryCompact编码有点相似
就拿上面那个例子:

```pb
message toServerData{
	required int32 ctype = 1;
	required string name =2;
    optional bytes httpdata=3;
	repeated string data =4;
}
```

- 有个`flag`开头表明类型
- 紧接着数据长度
- 紧接着数据本身
- **数组**类型`protobuf`不提供，只是在二进制中连续排序(在第一个元素`flag+length+data`后面每个元素都是`length+data`);

## 提供的一些连接方式


单向stream
双向stream
