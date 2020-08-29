---
title: OS basis
date: 2020-02-03 14:20
tags: OS
---

一些乱糟糟的笔记

<!--more-->

## 虚拟内存
段页式
以页为单位替换，以段为单位使用。


## 进程

资源调度的基本单位



结合golang调度器的一系列思考：

### 1. 为什么要多个进程

多任务处理

### 2. 为什么不可以在进程之间切换，还要加上线程？

回顾一下进程的定义，创建，销毁
- 定义

    就是运行期间的程序以及相关资源的集合体，多个进程可以共享一类资源，或者多个进程可以运行一个程序

- 创建

    linux上是由系统fork()(其实通过clone()来实现) init进程（或者现有进程）来创建一个新进程，注意这里会返回两次值，一次回到父进程，一次回到新的子进程，其**开销**其实就是复制父进程页表以及给子进程创建唯一task_struct，说多点这里，也不会一下子复制所有信息，采用copy on write，写时复制;

    - 详情
        同样是在linux下，fork()后 通过slab分配到task_struct()，其会有用到对象着色以及缓存着色（有点像进程池？？？)
        1. fork()内会调用copy_process()方法，首先会调用dup_task_struct()创建一个**内核栈**,**thread_info结构**和**task_struct**，这些值都与**当前进程(父进程)的值相同**

        2. 检查并确保新创建该子进程后，再检查当前进程数(默认short int 32768，当然你可以自己改/proc/sys/kernel/pid_max，忘了是不是这个了)

        3. 子进程就开始将一些值清零或者设为默认初始值来区分开，但都是一些统计信息(非继承的task_struct成员),task_struct的值大多数都没有变

        4. 子进程状态设置为**TASK_UNINTERRUPTABLE**，保证不被运行

        5. copy_flags()更新flags成员。清零PF_SUPERPRIV(是否是超级root)。设置PF_FORKNOEXEC()(表明进程还未被调用exec())

        6. 调用alloc_pid()为新进程分配一个有效pid

        7. 根据传给clone()的参数，copy_process()copy或者共享打开的文件，文件系统信息，信号处理函数，进程地址空间，命名空间等;

        8. 最后copy_process()返回一个指向子进程的指针;

    - 相关命令
        clone(CLONE_VM | CLONE_FS| CLONE_FILES| CLONE_SIGHAND, 0)
    - 额外
        还有一个vfork，不copy父进程的页面，其余同fork一样

- 执行
    调用exec()函数分配到相应的地址空间，然后将程序放入，
- 销毁
    调用exit()，这时候会将所有资源释放，父进程可以通过wait4()检查子进程是否被终结，还需要调用wait()或waitip(),否则子进程进入zombie状态





3. 进程之间如何调度

- 分清楚目的：
I/O bound 还是 CPU bound ?
- 手段
    1. 优先级？nice值？
    2. 时间片为基本单位
