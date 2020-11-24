---
title: Linux Cheat sheet
date: 2020-10-03 14:20
tags: Linux
---




- ps
From MANNUAL:

EXAMPLES
       To see every process on the system using standard syntax:
          ps -e
          ps -ef
          ps -eF
          ps -ely

       To see every process on the system using BSD syntax:
          ps ax
          ps axu

       To print a process tree:
          ps -ejH
          ps axjf

       To get info about threads:
          ps -eLf
          ps axms

       To get security info:
          ps -eo euser,ruser,suser,fuser,f,comm,label
          ps axZ
          ps -eM

       To see every process running as root (real & effective ID) in user format:
          ps -U root -u root u

       To see every process with a user-defined format:
          ps -eo pid,tid,class,rtprio,ni,pri,psr,pcpu,stat,wchan:14,comm
          ps axo stat,euid,ruid,tty,tpgid,sess,pgrp,ppid,pid,pcpu,comm
          ps -Ao pid,tt,user,fname,tmout,f,wchan

       Print only the process IDs of syslogd:
          ps -C syslogd -o pid=

       Print only the name of PID 42:
          ps -q 42 -o comm=

- 转换文件编码格式:
```shell
iconv -f [encoding] -t [encoding] sourcefile -o outputfile
```

- strace

一般用于**系统调用**的trace;


- tcpdump 
可以结合wireshark分析tcpdump



- sysctl -a
查看sys设置参数，包括ip设置，tcp设置等;
相关设置在`/etc/sysctl.conf`中;

在 `/proc/sys/net`也可以观察到;



- tcpdump
实时抓取当前连接，同wireshark相似

- tcp连接状态

```s
netstat -na | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'
```
- chmod
改变权限
权限组分为`拥有者`，`群组`，`其他用户`三种级别,分别为`RWS`，对应读、写、执行；

组成的权限为二进制的bit array，如下所示:
```s
r-- = 100
-w- = 010
--x = 001
--- = 000
```

注意：对于文件夹来说，`s`执行,是其读取`r`(ls),`w`写入(在里面进行创建删除更改等操作)的前提，没有`s`权限，


