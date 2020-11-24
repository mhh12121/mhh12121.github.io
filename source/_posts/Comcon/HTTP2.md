---
title: Notes about HTTP2
date: 2018-07-15 12:00
tags: networking
---

HTTP2 
<!--more-->

## 结构



## 相对于HTTP1.1 改变了啥子

### HTTP1.1提供的特性

HTTP1.1有一个关键的特性：keep-alive头字段
以前的HTTP协议都是规定 **每次请求都要建立一次连接（3次握手）**，更别提慢启动，拥塞控制这些捞东西了;

而HTTP1.1的keep-alive字段就可以保证 **一定时间内，同一域名多次请求数据，只建立一次HTTP请求**，其他请求可以复用这次建立的通道，这里一定时间可以通过工具配置（nginx，apache）

### HTTP1.1仍然有缺陷
1. 连接数过多，浏览器一般就会因为这个设置TCP连接限制;
一般是6-8个，假设是6,如果apache最大并发数为300,服务器每次承载最多只有300/6=50而已，多过这个数就要等待
2. 文件传输只能是**串行**的（在一个通道里面）

### HTTP2结构

HTTP2 传输的最小单位是 Frame（帧）。HTTP2 的帧包含很多类型：`DATA Frame`、`HEADERS Frame`、`PRIORITY Frame`、`RST_STREAM Frame`、`CONTINUATON Frame` 等。一个 HTTP2 请求/响应可以被拆成多个帧并行发送，每一帧都有一个 StreamID 来标记属于哪个 Stream。服务端收到 Frame 后，根据 StreamID 组装出原始请求数据;

如图:![tu](/img/http2header.png)


### HTTP2多路复用

1. 连接数过多的问题：
HTTP2在**同一域名**下所有连接都基于**流**，都在同一个连接，比如上面的并发为300,用HTTP2就能达到300;

2. 因为HTTP1.1传输的request和response都是**基于文本的**，所以所有数据必须按**顺序传输**才能保证可靠性（TCP）;
然而HTTP2前面说到利用的是**二进制数据帧**和**流**（TCP其实也是流），帧就是标识了数据的顺序！即不必要传完一个request，等到response再继续下一个;



## Question


###  why not use HTTP2 as download protocol?

[StackOverflow上有大佬提出](https://stackoverflow.com/questions/44019565/http2-file-download)

主要是两个点：
1. frame overhead



2. flow control

http2的server端会控制给每一个session和每一个在这个session里的stream的发送端口


### 造成的问题? Something not compatible with HTTP2?
1. JS文件合并:以前多个模块（文件）会合并成一个文件，当有模块要修改的时候，会全部上传一遍;
2. 多域名进行文件传输(domain sharding)，会导致**DNS解析时间过长**，**增加服务端压力**等