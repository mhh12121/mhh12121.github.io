---
title: Interview
date: 2020-09-30 12:00
tags: Interview
---

# 面试准备(后端)

## 语言go

### 1. 内存管理


#### 总体结构


- tcmalloc

- mspan

- mcache

- mheap

#### sync.Pool

- 环状列表

- victimCache (1.13)

### 2. 同步

#### 关键字及其用法原理

##### channel

##### select


##### sync.WaitGroup



#### 思想

- 名句




### 3. 垃圾回收

#### 三色波面推进(标记整理)

##### 具体相关字段

##### 流程

#### 比较其他垃圾回收器

- 引用计数


- 标记回收

- 标记回收整理

- 同步时回收

### 4. reflection

#### iface & eface


### 5. 继承(组合)

#### 一些常考的cases
string(), error(), read(), write() 

### 6. net包

#### epoll 


#### 优秀的连接池包
- fasthttp




## 计算机网络

总体七层


### 应用层常用

HTTP 80端口
##### 头，幂等设计

##### HTTP1.1, HTTP2 & HTTPS

##### 安全

- JWT
- HTTPS
- session


### 传输层
##### 总体


##### tcp
- 结构
    40 Bytes
    五元组<source ip, dest ip>
    标志位(RST,ACK,SYN,FIN)

- 面向连接，可靠
    1. 选择重传(快重传)
    2. 三次握手，四次挥手
    3. linux下常用默认设置

- 安全
    1. 校验码,纠错


##### udp
- 结构

    五元组
- 不可靠
    不用握手
- 安全
    1. crc只可以检测，不可以纠错



### 网络层

##### IP协议

MMU(max ) 1500 Bytes

##### DNS

- 43端口
- 权威，顶级，根
- 递归，迭代



##### ICMP 
trace route, ping

##### ARP

##### DHCP



### 链路层



##### MAC地址

##### BGP & 

- 域间路由,域内路由


- 链路毒逆转

### 物理层
略
- 几种光纤


## 数据库


### 持久型

MySQL, 5.7

#### 单机事务

- ACID

- 隔离性四个级别
    1. 串行
    2. repeatable Read, 保证写，可能有幻读(范围内的读取读取不到插入或者删除),一般是快照实现(MVVC)
    3. unrepeatable Read，

#### 基本常用语法

- explain 
- 

#### indexing

- 聚簇索引

- 非聚簇索引

#### 源码生成




### 缓存内存型

Redis
#### 概括

1. 内存跑

2. 单线程(网络部分), IO多路复用(epoll)

3. 优秀的数据结构

#### 缓存的几种应对

- 读多
    写时，先删除，再写
    读时，直接读

- 写多
    写后再更新

#### 基本结构(对外API)

string
- ziplist


list
- lpop, rpop,
    lpush, rpush
    llen
- 

dict
- ziplist

- hashmap

set
- dict

zset

- quicklist+ dict


#### aof & rdb
暂时略

#### replicated & sentinel & clusters

- offset复制

- raft log



### 操作系统(Linux)

#### 编码

大顶端，小顶端
ASCII

#### 进程，线程

-   1. 分配资源的基本单位
    2. task_struct{}

- 调度的基本单位


#### 调度

- 时间片轮转

- 优先级(nice 值 -20 ~ 19)

- 抢断式

#### 软中断和下半部



#### 内存分配



#### 文件

- 一切皆文件,都有一个fd


#### 硬盘

- 电梯法


### 计算机组成原理


#### 补码，原码，反码

正数,补码=原码,反码符号位除外全部取反



#### IEEE754

ps: 0.1+0.2!=0.3 问题

0.1无法用2进制准确表达


#### 总线BUS


#### Go中常用汇编(plan9)

- SP stack pointer
  PC 
  
常见命令:
ADD
SUB

MOV
SYMBOL+OFFSET(SP),伪SP
OFFSET(SP),真SP

JMP

### 分布式

CAP
分区容错
一致性
可用性

#### 负载均衡

1. 负载均衡算法
2. 健康检查(从IP层到HTTP层)
3. session的保持方式

#### 事务

2PC -> 3PC 

#### 共识算法


 Paxos -> MultiPaxos -> ZAB -> Raft


 ## Project

 1. 日志的输出kafka,处理; 包括失败下线，成功率，设计任务调度队列

 2. 缓存的任务调度队列，针对不同级别的任务队列,进程池的的大小来区分优先级;

 
 ## 项目

 1. job调度，使用工作池来限制其权重速率大小
 2. 限流，redis hmget，存储


 3. kafka，排除消息堆积:
 提交到broker的偏移量处开始消费。
        我在网上查阅了消息堆积和消息重复的一些原因，发现问题可能出现在kafka的poll()设置上。
        查阅kafka官网发现我用的那个版本的kafka主要有以下几个比较关键的指标:
        a. max.poll.records一次poll返回的最大记录数默认是500
        b. max.poll.interval.ms两次poll方法最大时间间隔这个参数，默认是300s
这次问题出现的原因为由于业务上下方的消息增量变多，导致堆积的消息过多，每一批poll()的处理都能达到500条消息，导致poll之后消费的时间过长。
 服务端约定了和客户端max.poll.interval.ms，两次poll最大间隔。
 如果客户端处理一批消息花费的时间超过了这个限制时间，broker可能就会把消费者客户端移除掉，提交偏移量又会报错。
 所以拉取偏移量没有提交到broker，分区又rebalance，下一次重新分配分区时，消费者会从最新的已提交偏移量处开始消费，这里就出现了重复消费的问题。
 而服务注册中心zookeeper以为客户端失效进行rebalance，因此连接到另外一台消费服务器，然而另外一台服务器也出现poll()超时，又进行rebalance…如此循环，才出现了一直重发消息，导致消息数量被消费后下降很慢。


rebalance:

consumer订阅topic中的一个或者多个partition中的消息，一个consumer group下可以有多个consumer，一条消息只能被group中的一个consumer消费。
consumer和consumer group的关系是动态维护的，并不固定，当某个consumer卡住或者挂掉时，该consumer订阅的partition会被重新分配给该group下其它consumer，用于保证服务的可用性。
为维护consumer和group之间的关系，consumer会定期向服务端的coordinator(一个负责维持客户端与服务端关系的协调者)发送心跳heartbeat，当consumer因为某种原因如死机无法在session.timeout.ms配置的时间间隔内发送heartbeat时，coordinator会认为该consumer已死，它所订阅的partition会被重新分配给同一group的其它consumer，该过程叫：rebalanced。

方案:

使用Kafka时，消费者每次poll的数据业务处理时间不能超过kafka的max.poll.interval.ms，可以考虑调大超时时间或者调小每次poll的数据量。
增加max.poll.interval.ms处理时长(默认间隔300s)
- max.poll.interval.ms=300

修改分区拉取阈值(默认50s,建议压测评估调小)

max.poll.records = 50

- 可以考虑增强消费者的消费能力，使用线程池消费或者将消费者中耗时业务改成异步，并保证对消息是幂等处理
- 不但要有消息积压的监控，还可以考虑做消息消费速度的监控（前后两次offset比较）


4. redis, 缓存的一致性，write aside，write through


5. failbackup,用资源换延迟，发送多个请求，有一个返回就用那个