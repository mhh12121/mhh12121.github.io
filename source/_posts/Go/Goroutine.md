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

GPM
G（goroutine）指的是go语言的goroutine（有些人叫它为协程，但其实跟coroutine有一点区别）

M（machine）指的就是OS原生线程，是真正调度资源的单位

P（Process）指的是go语言中的调度器，M就是用P才能调度G