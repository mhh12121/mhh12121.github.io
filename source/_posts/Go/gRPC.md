---
title: Something in gRPC 
date: 2019-06-29 14:40
tags: golang
---

常用的rpc框架

<!--more-->
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


## http2

见我自己的http2的文章，随便记了一点;
### server端针对不同的帧进行处理



### 超时问题

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

## proto
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
}
```

## 提供的一些连接方式


单向stream
双向stream
