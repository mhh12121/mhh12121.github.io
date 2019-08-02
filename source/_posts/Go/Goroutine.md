---
title: Goroutine Notes
date: 2019-07-09 01:00
tags: Golang
---

Goroutine 的模型，调度等，它与普通thread有何区别？先留个坑//todo
<!--more-->
## 为什么有这个东西？
传统OS自带的线程一个占栈1MB，明显大的过分，所以编程语言自身得另外实现一些小的线程


## 调度模型
一般来说多线程调度模型有 work-sharing 和 work-stealing模型

go采用了后者，可以看看有关[work-stealing的论文](http://supertech.csail.mit.edu/papers/steal.pdf)

架构：
GPM
- G（goroutine）指的是go语言的goroutine（有些人叫它为协程，但其实跟coroutine有一点区别，因为coroutine单纯在用户态使用）

- M（machine）指的就是OS原生线程，是真正调度资源的单位，M是idle或者syscall中，需要P的调度

- P（Process）指的是go语言中的调度器，M就是用P才能调度G


P在GOMAXPROCS中，所有的P被组织成一个数组，当GOMAXPROCS改变时会触发 stop the world来重新调整P 数组的长度
一些变量会从sched中分离出到P中

### Scheduler调度过程
当有新的Goroutine被创建或者是现存的goroutine更新为runnable状态，它会被push到当前P的runnable goroutine list里面，
当P完成了执行goroutine，它会
- 首先从自己的runnable g list里面pop一个goroutine，如果list是空的，它会随机选取其他P，并且偷取其list的一半runnable goroutine

当M 创建了新的goroutine，它要保证有其他M执行这个goroutine
同样的，如果M进入了syscall阶段，它也要保证有其他M可以执行这个goroutine


这里我们有两种方法：
我们可以立即block或unblock多个M，或使其自旋；
但这里会有性能损耗和花费不必要的cpu周期，方法是使用自旋而且burn CPU cycles
然而，这不应该影响在GOMAXPROCS=1的程序（command line，appengine这些）

自旋有两个等级：
1. 一个已经附着了一个P的idle M 会不断自旋寻找新的Goroutines
2. 一个已经附着了一个P的M 自旋等待其他可用的P 
以上中，最多有GOMAXPROCS个自旋的goroutines， 等级（1）的idle M 不会阻塞 即使有等级（2）的idle M；
当新的goroutine被创建或者M进入syscall或者M从idle变成busy，它会保证至少存在一个自旋M （或者所有P是busy），
1. 这就保证了不会有当前运行着的Goroutines被其他 M 运行
2. 也避免了过多的 M 同时 阻塞和释放阻塞


### 死锁检测和终止
当所有P是idle的时候进行检测（全局idle P的原子计数）

 



旋转态->不旋态的转换中，
可能和创建一个新的goroutine和创建一部分或其他需要unpark的工作线程 的时候发生竞态条件
如果转换和创建都失败，我们就可以以半静态cpu未充分利用结束；
goroutine 准备步骤是：提交一个goroutine去local queue，store-style memory 屏障，检查sched.nmspinning


不旋态->旋转态是： 减少nmspinning，store-style memory 屏障，检查新的work的所有per-P work queue
而且以上都不适用于global run queue