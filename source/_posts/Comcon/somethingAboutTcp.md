---
title: Something about Networking
date: 2019-07-02 15:01
tags: networking
---

记录了一些很基础的但经常被问到的问题
<!--more-->
### 1. 为什么多个tcp连接会比单个tcp连接快？（tmd面试傻了，居然没答出来这个）
一开始看见，这不是显而易见的吗？？？

后来发现，其实他想听到的答案是： 
1. tcp的流量窗口（拥塞控制）
如下图(盗图)：
![tcpwindow](/img/tcpWindow.jpg)
绿色为 发送者发送，且接收者acked
黄色为 发送者发送，接收者未确认（in-flight）
蓝色为 可用但为发送

cwnd= width（in-flight）+width（not sent）

发送速率：
**rate = cwnd / RTT byte/sec**
即发送速率在RTT（往返时延）一定的情况下，只受cwnd影响


2. 慢启动

tcp会进行**慢启动**直到丢包,每接到一个ack就会把窗口（cwnd）×2，当出现丢包的，有以下两种状况：
1. 接收者发送给发送者的ACK丢失，会导致 timeout
2. 发送者发送给接收者的数据丢失，发送者会收到接受者的重复ACK，如果收到三个重复的ACK，可以确认为丢包


3. 路由器（多个TCP有拥塞控制）
给出带宽为R，有K个连接经过
最后每个连接平均分的都会是 R/K

终上：

一个tcp连接很可能不能把当前路由的带宽都用完（），所以要多个tcp连接，这样才能最大保证速率

### 总结
回答问题，要从特么原理开始一步步推导来說，不能想当然

### 2. 各种握手挥手（我求求我自己把这些gdx记得滚瓜烂熟，每次都漏一点点）

#### TCP三次握手连接(three-way handshake)

直接上他🐎图：
![tcpconn](/img/TCPshakeFhand.jpg)

1. client发送server  
**SYN=1**（同步位，这种报文不能携带数据，但要**消耗一个序号**）
自己的序号 **Seq=client_w** 给 server （期间client从CLOSED到SYN-SENT状态，server从CLOSED到LISTEN状态）
2. server收到报文，把确认报文段中的SYN和ACK都设为1
 **SYN=1**(同理**要消耗一个序号**) 和 **ACK=1**
自己的序号 Seq=server_w, ack= client_w + 1 到client （期间client仍然是SYN-SENT状态，server从LISTEN状态到SYN-RCVD状态）
3. client收到确认报文后，还要给server发送确认收到。
把**确认ACK=1**,ack=server_w+1(之前SYN消耗了序号)
自己的序号Seq=client_w+1 (因为没有SYN，所以可以携带数据,**但如果不携带数据则不消耗序号**，这里携带了数据，所以Seq是上一次的序号+1);（client端进入established状态）
4. server收到确认报文后，进入established状态，全部连接完成


##### 三次握手能避免啥🐔儿东西呢（为啥两次不行？）

我们假设client给server发了个连接请求，但是请求丢失，然后client再发一个，server收到这个然后建立连接，这里面client发送了两个请求
会有以下情况出现：

client发出的第一个请求没有丢，只是网络阻塞;
但接下来，server收到后以为是新的一个连接，server返回一个确认报文，但client收到server确认报文会发现自己并没有建立连接请求，所以会忽视server的确认报文，也不会向server发送数据
但server会**一直开着连接等client的数据，白白浪费资源!**
（但采用三次握手的话就没得事，server端没有接到client的第三次确认，就知道client没有要建立连接）


#### TCP四次挥手

![tcpfour](/img/TCPgoodbye.jpg)
1. client 发送 
**FIN=1**（终止位，跟同步位一样都在header里面，<del>自己特喵去看</del>下面给你画一个算了,但这里注意，**无论带不带数据，这厮都要消耗一个序号！**）
**seq=client_u** （这里序号是**前面一个传送过的数据最后一个字节的序号+1**）;(client进入FIN-WAIT-1状态)

2. server收到释放报文返回 
**ACK=1**
**Seq=server_u**(同理，也是前面一个传送过的数据的最后一个字节序号+1);(server进入CLOSE-WAIT状态，但实际上是个半关闭状态，server->client方向的连接保持，但client->server已经没有**数据**要传输了)

3. client收到server的确认后，进入FIN-WAIT2状态，等待server的连接释放报文

4. 如果server没有数据要发给client了（server持续发送数据到client，也可以不发），
携带报文 **FIN=1**， ACK=1
**seq=server_u2**(新，可能发送了一些数据)
**ack=client_u+1**（重复上次已发送的确认号）
(server进入LAST-ACK状态)

5. (紧接3)client收到释放报文后，返回确认报文：
ACK=1 
确认号**ack=server_u2+1**
序号**seq=client_u+1** （前面1的FIN报文消耗了一个序号client_u）
然后client进入TIME_WAIT状态，然后经过2MSL（RFC 793设为2mins，但其实应该要用更小的数值）再进入CLOSED状态
client撤销相应的TCB（传输控制块）后，结束连接

##### 为啥要等2MSL
1. 保证client发送的最后一个ACK报文能够到达server;
因为这个报文可能会丢失，然后处在LAST-ACK的server收不到已发送的FIN+ACK报文的确认。正常情况下，server会重传FIN+ACK报文，client接着重传最后一个ACK报文，重新计时;
但如果client不等待，发送完ACK报文直接释放连接进入CLOSED，client就没法收到server重传的FIN+ACK报文，也不会再发送一次确认报文（因为关闭了啊！），这样server就不会正常进入CLOSED状态

2. 防止出现 ‘已失效的连接请求报文段’(如同三次握手时间的client第一次发出的报文);
client发送完最后一个ACK报文段后，经过2MSL，可以确定本连接持续时间内的所有报文段都从网络消失，这样就不会出现旧的连接报文段了

##### 为啥连接用三次，断开要四次呢？
- 主要区别还是在连接时，server收到连接请求后可以直接返回SYN+ACK报文；
- 但是断开时，server收到了client的FIN报文后，可能还有数据没有传完，只能先发回一个ACK，等到最后数据都传完了才会发FIN，所以为了避免没传完数据就关闭了的情况，只能加多一次连接;

 
PS：server的cclosed状态要比client的要早一点
MSL>=TTL

连接当中如果用HTTPS，参考之前写的{% post_link LoginSecurity https %}



##### keepalive timer
2MSL是被client的一个TIME-WAIT timer设置的，实际上TCP还有一个keepalive timer，存在header里面的keep-alive

### 3. DNS协议

首先这🐔是**应用层**协议！！！
功能主要就是**把域名转换为IP地址**
然后，主要是在**UDP**上面跑，端口**53**
长度最多是**512Bytes**，若过多要用 ![EDNS](https://en.wikipedia.org/wiki/Extension_mechanisms_for_DNS)




### 4. URI 和 URL 和域名

URI（统一资源标识符） 包括 URL（统一资源定位符）和URN（统一资源名称）

URI通用模式：
```
scheme：[// [user：password @] host [：port]] [/] path [？查询] [#片段]
```

URL：主要用于连接网页或部分部件，借助访问的协议（http，ftp等）来检索定位资源位置
URL包含了
1. 访问资源的协议
2. 服务器位置
3. 端口
4. 资源在服务器上的位置
5. 片段标识符（就是锚点#something）

域名（domain name）指的是任一主机或路由器连接在因特网上都有**唯一**的 **层次结构的名字**,只是一个**逻辑概念**


### 5. 浏览器输入url发生了什么 （个人感觉按照这个🐔来复习会比较好）


1. 输入URL，浏览器会解析url，这里面是通过DNS域名解析，找到url对应的服务器地址，其中可能会从HOSTS文件找
2. 找到主机地址，就会连接主机，这里面tcp三次握手，发送HTTP请求
然后封装发送的包，http包放在tcp包里，tcp包放在IP包里，层层往下，每一层会通过网管（gateway）
2. 到了IP协议处会将其通过ARP解析出相应的在链路层的物理地址（通过路由器），在网络层和以上使用都是IP地址，以下都是硬件地址了，所以数据链路层看不见IP地址了
3. 网络层通过物理地址找到路由器，把层层封装的包通过网桥（或桥接器等）等发送到链路层上，当前只能看见MAC帧，链路层负责开始传输数据了
4. 物理层只是把传输数据 变成电信号真正地传到网线上