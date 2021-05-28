---
title: Linux Cheat sheet
date: 2020-10-03 14:20
tags: Linux
---




- ps
From MANNUAL:

EXAMPLES
```s
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
```
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


- free 
相对与`top` 命令来说，free命令是准确展示内存和cache的，因为linux本身的机制，有些内存来不及回收，当做`cache`存留在内存中，所以top命令看到的内存大小是不一定正确的，可能有大部分cache占用了;

- 查找程序莫名被killed的原因

1. `/var/log`下查找，每天的log中过滤killed process等字眼即可
   可以通过`egrep`(同grep -e，只不过匹配原则不一样):
```s
egrep -i 'killed process' /var/log/messages
```
2. 或者`journalctl`:
```s
journalctl -xb | egrep -i 'killed process'
```

```s
//
journalctl -k
//一段范围
journalctl --since="2018-09-21 10:21:00" --until="2018-09-21 10:22:00"
//日志占用空间
journalctl --disk-usage
```

3. 或者`dmesg`来查看开机信息:


- time_wait过多

即连接过多,且瞬时断开的较多，一般是短连接断开;
那样就有对策：加大点连接总数（端口数）、增大连接时长、重用socket，及时回收或者上游确定是否要用短连接，可否改为长连接?

`net.ipv4.tcp_syncookies = 1`

表示开启SYN Cookies。当出现SYN等待队列溢出时，启用cookies来处理，可防范少量SYN攻击，默认为0，表示关闭；

`net.ipv4.tcp_tw_reuse = 1`

表示开启重用。允许将TIME-WAIT sockets重新用于新的TCP连接，默认为0，表示关闭；

`net.ipv4.tcp_tw_recycle = 1`

表示开启TCP连接中`TIME-WAIT` sockets的快速回收，默认为0，表示关闭。

系统tcp_timestamps缺省就是开启的，所以当`tcp_tw_recycle`被开启后，实际上这种行为就被激活了.如果服务器身处NAT环境，安全起见，通常要禁止`tcp_tw_recycle`，至于TIME_WAIT连接过多的问题，可以通过激活tcp_tw_reuse来缓解。

`net.ipv4.tcp_max_tw_buckets = 5000`

表示系统同时保持TIME_WAIT套接字的最大数量，如果超过这个数字，TIME_WAIT套接字将立刻被清除并打印警告信息。默认为180000，改为 5000。对于Apache、Nginx等服务器，上几行的参数可以很好地减少TIME_WAIT套接字数量，但是对于Squid，效果却不大。此项参数可以控制TIME_WAIT套接字的最大数量，避免Squid服务器被大量的TIME_WAIT套接字拖死。

`net.ipv4.tcp_max_syn_backlog = 8192`

表示SYN队列的长度，默认为1024，加大队列长度为8192，可以容纳更多等待连接的网络连接数。

`net.ipv4.tcp_keepalive_time = 1200`

表示当keepalive起用的时候，TCP发送keepalive消息的频度。缺省是2小时，改为20分钟。

`net.ipv4.ip_local_port_range = 1024 65000`

表示用于向外连接的端口范围。缺省情况下很小：32768到61000，改为1024到65000。