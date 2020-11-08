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

