---
title: Notes about Raft and Paxos
date: 2018-07-10 16:50
tags: Distributed
---

复习一些分布式理论: Raft和Paxos(暂时只讨论算法情况，工程情况留坑)
<!--more-->
## PAXOS

paxos实际上是一系列的协议，有basic-paxos, multi-paxos

目的：让多个参与者达成一致

**一个原则**：参与者如果达成一致，这个一致的观点在传递中永不会改变
### basic-Paxos

#### 一些角色和名词
##### client
只是负责发起request到分布式系统并等待相应
##### Proposer
（发起者）
它负责从client 发起请求，尝试让Acceptor来同意这个请求，也会在冲突发生的时候作为一个合作者来使协议继续执行（propser可以不断发起提议，不用等提议完成）

##### Acceptor（Voter）
（投票者）
多个Acceptor组成一个Quorums（法定多数），所有发向一个Acceptor的必须要发给Acceptor的一个Quorum，任何从一个Acceptor来的信息会被忽略，除非能接受到一个从这个Quorum里所有的Acceptor来的拷贝？？？

##### Quorums
（多数派）
Quorums被定义为某些Accetors的集合的子集，即任何两个Quorums都会有至少一个Acceptor交集，而且Quorums总会是包含大部分的Acceptor;比如：有一个Accetpros的集合{A，B，C，D}，大部分的Quorums可以是任意三个Acceptor组成：{A，B，C}，{A，C，D}等等
而经常Acceptor还会携带不同的权值，但保证的是Quorums所包含的所有Acceptors的权值和都会**大于**总的权值和（所有的Acceptors）的一半;
##### Learner
只是作为协议的复制因素，一旦client的request被Acceptor同意，learner就会开始执行这个request以及返回给client一个响应;
一般为了提高协议的可用性，可以额外增加多个learner


#### 基本流程
每个basic-paxos的实例（或执行者）决定于一个输出值。该协议在多轮通信中进行;
成功的有两轮，每轮有a，b，我们假定是在一个异步模型里面，即一个processor可能在第一轮，另一个可能在第二轮

##### Phase 1：
###### 1a：Prepare
 Proposer创建一个信息，我们称为Prepare，附带一个唯一标识数字 n 。而且当前的 n 要大于这个Proposer之前发的信息附带的所有 nX;
 然后，Porcessor把这个带有 n 的Prepare信息（这里只带有n，实际上它不用带其他信息，比如提议内容）发到在一个Quorum的Acceptor上（哪些Acceptor在Quorum里面是由Proposer决定的[跳到如何决定]())，
 如果Proposer不能与至少一个Quorum通信则不应该初始化Paxos
 

###### 1b：Promise
（两个承诺，一个应答）

 每一个Acceptor会等待Proposer的Prepare信息，如果一个Acceptor收到了，它会查看附带的n标识，会有两种情况：
   1. 如果**n比之前收到的所有proposal n都大**（从任何Proposer收到的），这个Acceptor一定要返回一个信息Promise给对应的Proposer，然后会忽视所有未来的附带标识小于n的提议（信息）。
   如果这个Acceptor以前接收过**其他提议**，返回给这个Proposer的response一定要包含之前的提议编号m，和对应的值w;（这里Acceptor还会持久化提议）
   2. 如果**n出现<=之前收到的任何一个proposal n**，Acceptor能忽略这个提议;
   为了优化，还是可回一个denial response，提示Proposer可以停止创建带有n的提议了

##### Phase 2：
###### 2a： Accept

  如果一个Proposer收到了的从Quorum返回的response，
  1. 如果未超过一半的Acceptors同意，提议失败
  2. 如果超过了一半：
    如果**所有**Acceptor都没有收到内容（null），即可发起proposal的内容，然后带上当前的**proposal n**，向所有Acceptor再发送提议;
    如果有**部分**Acceptor接到过内容，会从所有接受过的内容中，**选择proposal n最大**的内容作为真正的内容，提议编号仍然为n，但这时Proposer就不能提议自己的内容，只能信任Acceptor通过的内容;
  

###### 2b：Accepted
  如果Acceptor接收到了提议后，他必须遵循：如果有且仅有不违背 **Phase1b（两个承诺）**情况下（即该提议n等于之前Phase1保存的编号），记录下(持久化）**当前proposal n 和内容**;
  最后Proposer收到Quorum返回的Accept response后，形成决议

#### 图解在算法异常的情况下（工程下更加复杂，先占坑）：

1. 没有失败的情况下

有一个client，1一个proposer，3个Acceptor，和两个learnner;

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

2. Acceptor失败的情况

1. Quorum里面有一个Acceptor失败了，所以整个Quorum大小会变成2，整个paxos还是成功的（此时Quorum数目> Acceptors/2)
2. 如果有两个或三个失败，则直接返回失败;
或Proposer重新发起提议（内容一样，提议号n+1）但是，对第一次已经成功接收的acceptor不会修改，其余上一次失败的acceptors才会接收提议

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

3. Leaner也可能会失败

虽然有一个learner失败了，但整个paxos还是成功的

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

4. Proposer失败

在Proposer接收到proposal值（提议内容）之后，发回的Accept阶段失败了，只有一个Acceptor接到提议;
同时，一个新的Proposer被选举出来：


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

5. 多个Proposer冲突 ！！！！
锁住了对方的Accept，导致prepare的值全部作废
这里问题大了。。。。（其实就引出了multi-paxos）
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


### Multi-Paxos
multi-Paxos将集群分为两种状态




以上参考 *** ![wiki-Paxos](https://webcache.googleusercontent.com/search?q=cache:zXcryn67tFcJ:https://en.wikipedia.org/wiki/Paxos_(computer_science)+&cd=2&hl=en&ct=clnk) ***//这两天不知道为啥上不去？只能看cached版（哭




## Raft
Raft可以看成是multi-paxos版本(其实有很多实现，没有统一)

比较简单，可以从几个方面进行理解，为**主从筛选(leader election)**和 **日志复制(log replication)**,**安全性(Safety)**，**减少状态(state space reduction)**






![大牛的证明（中文）](https://github.com/maemual/raft-zh_cn/blob/master/raft-zh_cn.md)






