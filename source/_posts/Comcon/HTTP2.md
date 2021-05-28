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

#### HPACK

头部压缩
HTTP /1 的请求头较大，而且是以纯文本发送，HTTP/2 对消息头进行了压缩，采用的是 `HPACK` 算法；
能够节省消息头占用的网络流量，其主要是在两端建立了索引表，消息头在传输时可以采用索引，而 HTTP/1.x 每次请求，都会携带大量冗余头信息，浪费了很多带宽资源。




#### Frame


HTTP2 传输的最小单位是 Frame（帧）。HTTP2 的帧包含很多类型：`DATA Frame`、`HEADERS Frame`、`PRIORITY Frame`、`RST_STREAM Frame`、`CONTINUATON Frame` 等;

这也是多路复用的关键: 一个 HTTP2 请求/响应可以被拆成多个帧并行发送，每一帧都有一个 StreamID 来标记属于哪个 Stream。服务端收到 Frame 后，根据 StreamID 组装出原始请求数据,同一个 stream 内 frame 必须是有序的,`SETTINGS_MAX_CONCURRENT_STREAMS` 控制着最大并发数(并且这个设置仅适用于接收设置的对端);

如图:![tu](/img/http2header.png)


- 所有帧都以固定的 9 字节头开头，后跟可变长度的有效载荷，组成如下：
- 长度：帧有效负载的长度表示为无符号的 24 位整数
- 类型：8 位类型的帧，帧类型确定帧的格式和语义
- 标志：为特定于帧类型的布尔标志保留的 8 位字段
- R：保留的 1 位字段。该位的语义未定义
- 流标识符：流标识符，表示为无符号 31 位整数，客户端发起流标识符必须时**奇数**，服务端发起的流标识符必须是**偶数**,流标识符零(0x0)用于**连接控制消息**,零流标识符不能用于建立新的 stream 流

24+8+8+1+31=9B*8=72bit

>> 常用的标志位有 `END_HEADERS` 表示头数据结束，相当于 HTTP/1 里头后的空行（“\r\n”），`END_STREAM` 表示单方向数据发送结束（即 EOS，End of Stream），相当于 HTTP/1 里 Chunked 分块结束标志（“0\r\n\r\n”）

### HTTP2多路复用

1. 连接数过多的问题：
HTTP2在**同一域名**下所有连接都基于**流**，都在同一个连接，比如上面的并发为300,用HTTP2就能达到300;

2. 因为HTTP1.1传输的request和response都是**基于文本的**，所以所有数据必须按**顺序传输**才能保证可靠性（TCP）;
然而HTTP2前面说到利用的是**二进制数据帧**和**流**（TCP其实也是流），帧就是标识了数据的顺序！即不必要传完一个request，等到response再继续下一个;

3. 个人认为本质上，HTTP2为了解决`head of line blocking`这种问题，可以理解为将 HTTP1.1 这种`一条通道`的 切割成`多个通道`(现在大多数的stream数目为`100`)，每个通道放不同的stream，达到了`multiplexing`的效果, 上面的flow control就在应用层上细化了粒度，管理这样不同`通道`的速度（TCP可以理解为为管理整个连接）


## Question


###  why not use HTTP2 as download protocol?

[StackOverflow上有大佬提出](https://stackoverflow.com/questions/44019565/http2-file-download)

主要是两个点：
1. frame overhead

HTTP1.1 每次传输实际都是content的bytes，但是HTTP2每次传输`DATA`frame额外的`9 bytes`的header


2. flow control

http2的`server`端会控制给每一个session和每一个在这个session里的stream的发送窗口,默认是`65535`bytes，如果要开启下载，一定要扩大该窗口;
然而问题来了，扩大该窗口是`client`端发送`WINDOW_UPDATE`进行扩大的，万一遇到延迟，这个传输无效，那么窗口只可以保持默认值大小


### 造成的问题? Something not compatible with HTTP2?
1. JS文件合并:以前多个模块（文件）会合并成一个文件，当有模块要修改的时候，会全部上传一遍;
2. 多域名进行文件传输(domain sharding)，会导致**DNS解析时间过长**，**增加服务端压力**等