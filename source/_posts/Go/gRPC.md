---
title: Something in gRPC 
date: 2019-06-29 14:40
tags: golang
---

常用的rpc框架，我们先从它的proto开始了解吧//todo
<!--more-->

我们这里只讨论go语法，语言方面其他都是大同小异
## proto

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
```shell
protoc --proto_path=/mypath --go_out=plugins=grpc:. *.proto //当前在mypath路径下，用grpc模式生成proto文件
```

生成 **data.pb.proto**

-----------这些就直接参照手册理解吧-----------------  
这里还是记录下这厮的解码方式吧，参考
>> 《数据密集型系统设计》

protobuf的的编码其实跟 thrift的BinaryCompact编码有点相似
就拿上面那个例子:

```
message toServerData{
	required int32 ctype = 1;
	required string name =2;
    optional bytes httpdata=3;
}
```

## 提供的一些连接方式


单向stream
双向stream
