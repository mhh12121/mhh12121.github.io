---
title: Notes about Raft and Paxos
date: 2018-07-10 16:50
tags: Distributed
---

复习一些分布式理论: Raft和Paxos(暂时只讨论算法情况，工程情况留坑)
<!--more-->

## PAXOS

paxos实际上是一系列的协议，有basic-paxos, multi-paxos

目的：让多个参与者达成一致，共识

**一个原则**：参与者如果达成一致，这个一致的观点在传递中永不会改变
### basic-Paxos

#### 一些角色和名词
1. client

只是负责发起request到分布式系统并等待相应

2. Proposer（发起者）
每个Proposer也是一个Acceptor
它负责从client 发起请求，尝试让Acceptor来同意这个请求，也会在冲突发生的时候作为一个合作者来使协议继续执行（propser可以不断发起提议，不用等提议完成）

3. Acceptor（Voter）
每个Acceptor也是一个Proposer
**多个Acceptor**组成一个Quorums（法定多数），所有发向一个Acceptor的必须要发给Acceptor的一个Quorum,不会同意比自己以前接受过的提案编号小的提案，任何从一个Acceptor来的信息会被忽略，除非能接受到一个从这个Quorum里所有的Acceptor来的拷贝？？？

4. Quorums（多数派）

Quorums被定义为某些Accetors的集合的子集，即任何两个Quorums都会有至少一个Acceptor交集，而且Quorums总会是包含大部分的Acceptor;比如：有一个Accetpros的集合{A，B，C，D}，大部分的Quorums可以是任意三个Acceptor组成：{A，B，C}，{A，C，D}等等
而经常Acceptor还会携带不同的权值，但保证的是Quorums所包含的所有Acceptors的权值和都会**大于**总的权值和（所有的Acceptors）的一半;

5. Learner

只是作为协议的复制因素，一旦client的request被Acceptor同意，learner就会开始执行这个request以及返回给client一个响应;
一般为了提高协议的可用性，可以额外增加多个learner


#### 基本流程
每个basic-paxos的实例（或执行者）决定于一个输出值。该协议在多轮通信中进行;
成功的有两轮，每轮有a，b，我们假定是在一个异步模型里面，即一个processor可能在第一轮，另一个可能在第二轮

首先进行典型的2PC 协议

Phase 1：
1.  1a：Prepare

 Proposer创建一个信息，我们称为**Prepare**，附带一个**唯一标识数字 n** 。
  - 而且当前的 n 要大于这个Proposer之前发的信息附带的所有 nX;
  - 每次n = ++maxProposal;
  - Porcessor把这个带有 n 的Prepare信息（这里只带有n，实际上它不用带其他信息，比如提议内容）发到在一个Quorum的Acceptor上（哪些Acceptor在Quorum里面是由Proposer决定的???)

 如果Proposer不能与至少一个Quorum通信则不应该初始化Paxos
 

2.  1b：Promise
（两个承诺，一个应答）

 每一个Acceptor会等待Proposer的Prepare信息，如果一个Acceptor收到了，它会查看附带的n标识，会有两种情况：
   1. 如果**n比之前收到的所有proposal n都大**
      - 这个Acceptor一定要返回一个信息Promise给对应的Proposer，然后会忽视所有未来的附带标识小于n的提议（信息）。
      - 如果这个Acceptor以前接收过**其他提议**，返回给这个Proposer的response一定要包含之前的提议编号m，和对应的值w;（这里Acceptor还会**持久化**该proposal值）

   2. 如果**n出现<=之前收到的任何一个proposal n**
      - 则不接受proposal n<=当前请求的prepare请求，Acceptor能忽略这个提议;
      - 为了优化，还是可回一个denial response，提示Proposer可以停止创建带有n的提议了，且该response可以带上一个当前Acceptor的promiseProposal，以便Proposer的更新

Phase 2：
1.  2a： Accept(propose)

  如果一个Proposer收到了的从Quorum返回的response，
  1. 如果未超过一半的Acceptors同意，提议失败
  2. 如果超过了一半：
    - 如果**所有**Acceptor都没有收到内容（null），即可发起proposal的内容，然后带上当前的**proposal n**，向所有Acceptor再发送提议;

    - 如果有**部分**Acceptor接到过内容，会从所有接受过的内容中，**选择proposal n最大**的内容作为真正的内容，提议编号仍然为n，但这时Proposer就不能提议自己的内容，只能信任Acceptor通过的内容;
  

2. 2b：Accepted
  如果Acceptor接收到了提议后，他必须遵循：
    - 如果有且仅有不违背 **Phase1b（两个承诺）**情况下（即该提议n等于之前Phase1保存的编号），记录下(**持久化**）**当前proposal n 和内容**;
  最后Proposer收到Quorum返回的Accept response后，形成决议

#### 图解在算法异常的情况下（工程下更加复杂，先占坑）：

1. 没有失败的情况下

有一个client，1一个proposer，3个Acceptor，和两个learnner;
```
Client   Proposer      Acceptor     Learner
   |         |          |  |  |       |  |
   X-------->|          |  |  |       |  |  Request
   |         X--------->|->|->|       |  |  Prepare(1)
   |         |<---------X--X--X       |  |  Promise(1,{Va,Vb,Vc})
   |         X--------->|->|->|       |  |  Accept!(1,V)
   |         |<---------X--X--X------>|->|  Accepted(1,V)
   |<---------------------------------X--X  Response
   |         |          |  |  |       |  |
Here, V is the last of (Va, Vb, Vc).
```
2. Acceptor失败的情况

1. Quorum里面有一个Acceptor失败了，所以整个Quorum大小会变成2，整个paxos还是成功的（此时Quorum数目> Acceptors/2)
2. 如果有两个或三个失败，则直接返回失败;
或Proposer重新发起提议（内容一样，提议号n+1）但是，对第一次已经成功接收的acceptor不会修改，其余上一次失败的acceptors才会接收提议
```
Client   Proposer      Acceptor     Learner
   |         |          |  |  |       |  |
   X-------->|          |  |  |       |  |  Request
   |         X--------->|->|->|       |  |  Prepare(1)
   |         |          |  |  !       |  |  !! FAIL !!
   |         |<---------X--X          |  |  Promise(1,{Va, Vb, null})
   |         X--------->|->|          |  |  Accept!(1,V)
   |         |<---------X--X--------->|->|  Accepted(1,V)
   |<---------------------------------X--X  Response
   |         |          |  |          |  |
```
3. Leaner也可能会失败

虽然有一个learner失败了，但整个paxos还是成功的
```
Client Proposer         Acceptor     Learner
   |         |          |  |  |       |  |
   X-------->|          |  |  |       |  |  Request
   |         X--------->|->|->|       |  |  Prepare(1)
   |         |<---------X--X--X       |  |  Promise(1,{Va,Vb,Vc})
   |         X--------->|->|->|       |  |  Accept!(1,V)
   |         |<---------X--X--X------>|->|  Accepted(1,V)
   |         |          |  |  |       |  !  !! FAIL !!
   |<---------------------------------X     Response
   |         |          |  |  |       |
```
4. Proposer失败

在Proposer接收到proposal值（提议内容）之后，发回的Accept阶段失败了，只有一个Acceptor接到提议;
同时，一个新的Proposer被选举出来：

```
Client  Proposer        Acceptor     Learner
   |      |             |  |  |       |  |
   X----->|             |  |  |       |  |  Request
   |      X------------>|->|->|       |  |  Prepare(1)
   |      |<------------X--X--X       |  |  Promise(1,{Va, Vb, Vc})
   |      |             |  |  |       |  |
   |      |             |  |  |       |  |  !! Leader fails during broadcast !!
   |      X------------>|  |  |       |  |  Accept!(1,V)
   |      !             |  |  |       |  |
   |         |          |  |  |       |  |  !! NEW LEADER !!
   |         X--------->|->|->|       |  |  Prepare(2)
   |         |<---------X--X--X       |  |  Promise(2,{V, null, null})
   |         X--------->|->|->|       |  |  Accept!(2,V)
   |         |<---------X--X--X------>|->|  Accepted(2,V)
   |<---------------------------------X--X  Response
   |         |          |  |  |       |  |
```
5. 多个Proposer冲突 ！！！！
锁住了对方的Accept，导致prepare的值全部作废
这里问题大了。。。。（其实就引出了multi-paxos）
```
Client   Leader         Acceptor     Learner
   |      |             |  |  |       |  |
   X----->|             |  |  |       |  |  Request
   |      X------------>|->|->|       |  |  Prepare(1)
   |      |<------------X--X--X       |  |  Promise(1,{null,null,null})
   |      !             |  |  |       |  |  !! LEADER FAILS
   |         |          |  |  |       |  |  !! NEW LEADER (knows last number was 1)
   |         X--------->|->|->|       |  |  Prepare(2)
   |         |<---------X--X--X       |  |  Promise(2,{null,null,null})
   |      |  |          |  |  |       |  |  !! OLD LEADER recovers
   |      |  |          |  |  |       |  |  !! OLD LEADER tries 2, denied
   |      X------------>|->|->|       |  |  Prepare(2)
   |      |<------------X--X--X       |  |  Nack(2)
   |      |  |          |  |  |       |  |  !! OLD LEADER tries 3
   |      X------------>|->|->|       |  |  Prepare(3)
   |      |<------------X--X--X       |  |  Promise(3,{null,null,null})
   |      |  |          |  |  |       |  |  !! NEW LEADER proposes, denied
   |      |  X--------->|->|->|       |  |  Accept!(2,Va)
   |      |  |<---------X--X--X       |  |  Nack(3)
   |      |  |          |  |  |       |  |  !! NEW LEADER tries 4
   |      |  X--------->|->|->|       |  |  Prepare(4)
   |      |  |<---------X--X--X       |  |  Promise(4,{null,null,null})
   |      |  |          |  |  |       |  |  !! OLD LEADER proposes, denied
   |      X------------>|->|->|       |  |  Accept!(3,Vb)
   |      |<------------X--X--X       |  |  Nack(4)
   |      |  |          |  |  |       |  |  ... and so on ...
```

### Multi-Paxos
multi-Paxos将集群分为两种状态




以上参考 *** ![wiki-Paxos](https://webcache.googleusercontent.com/search?q=cache:zXcryn67tFcJ:https://en.wikipedia.org/wiki/Paxos_(computer_science)+&cd=2&hl=en&ct=clnk) ***




## Raft
解决一致性问题的三个子问题

比较简单，可以从几个方面进行理解，为
- 主从筛选(leader election)
- 日志复制(log replication)
- 安全性(Safety,leader变更时)

### 主从筛选

保证任何时期最多只有一个leader，leader节点有所有已提交的日志


### 日志复制

指的是Raft保证每个副本日志append的**连续性**

leader会为每个follower维护一个nextIndex，表示leader给各个follower发送的下一条log entry在log中的index。

如果日志append的时候前一个日志还没有append，则必须等到前一个日志append后才能append。也就是说不同副本上相同index的日志，只要term相同，那么这两条日志必然相同，且这之前的日志必然也相同。




### 安全性

Leader只能附加的原则，只允许leader commit被大部分append的log entry;
即如果当前的log已被commit，证明在这之前的所有log都被提交

#### 节点错误处理

宕机等有几种情况：

1. Followers 或 Candidates
Followers 或者candidates 崩溃, 解决办法只需要leader不断重试发送请求即可， 再不行就重启该崩溃的服务器，就能收到AppendEntriesRPC和Requestvotes请求

2.  Leader
但是Leader 崩溃，如果直接重启服务器：
在完成了一次RPC发送但没有接收response的时候重启，他会再次收到一个相同的RPC，但是因为Raft RPC是幂等的，所以这个没有关系，举个例子，
如果一个follower接到了AppendEntries请求，而且发现这个请求的log entries已经在自己的log里面了，他会忽略这个请求

3. 网络分区，多数派的leader照常工作，少数派按照道理是不能选出leader，所以没有工作，恢复后少数派没有最新的log，所以肯定是成为follower，leader只需要appendEntries回复这些followers的log即可

#### 时间和可用性
时间指timing，这里尤其是leader的选举，是时间敏感的，
Raft有个公式，可以保证能选举和维护一个稳定的leader:

>> broadcastTime<< electionTimeout << MTBF

broadcastTime: 广播的平均时间（某个节点）
electionTimeout: 选举的时间
MTBF: 前一次失败和这一次失败间隔的平均时间（某个节点）

broadcastTime<< electionTimeout 可以保证leader发送heartbeat给其他followers，防止他们开始选举

electionTimeout<< MTBF 可以保证系统平稳进步？


一般来说， broadcastTime只会在0.5ms ~ 20ms取决于持久化的技术，所以相应的，electionTimeout就会是在10ms~500ms之间， 比较重要的节点的MTBF则取几个月或更多

### 节点成员改变

我们之前都是假定节点配置是不变的，但实际上当有server crash的时候就需要替换他们
虽然我们可以把所有接地都下线，更新配置，然后上线，但这明显有问题

1. 保证配置的安全， 一个term期间，在有可能有两个leader被选举的时候进行配置的传输，但在转移配置的时候，不能保证所有server都能第一时间拿到新的配置，
可能会导致节点分裂成两部分，两个独立的大多数

为了保证安全，只能分为 **两步提交** 
一些系统就会做 第一步：关闭旧的配置，第二开启新配置

在raft里面，节点第一次切换到新的配置，我们称之为joint consensus（交叉共识），一旦交叉共识被commit，整个系统就都转到了新的配置上

#### Joint consensus
交叉共识包含
log entries会被复制到新旧配置中的所有server
任何节点在任一配置中都能被当做leader
 共识（这里针对选举和提交）要分开的大多数配置通过 （旧的大多数和新的大多数）


## Raft缺点

每次都是串行投票,串行apply

[大牛的证明（中文）](https://github.com/maemual/raft-zh_cn/blob/master/raft-zh_cn.md)



