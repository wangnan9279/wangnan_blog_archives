---
title: Linux常用命令之ps
tags: [Linux]
date: 2017-03-22 16:43:12
categories: Linux
link_title: linux-ps
---
> 查看当前用户的活动进程

<!--more-->

# 参数
如果加上参数可以显示更多的信息，如-a，显示所有用户的进程
　
**ps -ax** ：tty值为“?”是守护进程，叫deamon 无终端，大多系统服务是此进程，内核态进程是看不到的
      
**ps -axf** ：看进程树，以树形方式现实进程列表敲 ，init是1号进程，系统所有进程都是它派生的，杀不掉
      
**ps -axm**：会把线程列出来。在linux下进程和线程是统一的，是轻量级进程的两种方式。

**ps -axu**：显示进程的详细状态。

**ps -vsz**：说此进程一共占用了多大物理内存。

**ps -rss**：请求常驻内存多少


# 查看线程
其实linux没有线程，都是用进程模仿的
1. ps -ef f
用树形显示进程和线程，比如说我想找到proftp现在有多少个进程/线程，可以用
$ ps -ef f | grep proftpd
    


    nobody 23117 1 0 Dec23 ? S 0:00 proftpd:   (accepting   connections)   
    jack 23121 23117 0 Dec23 ? S 7:57 \_ proftpd: jack - ftpsrv:   IDLE
    jack 28944 23117 0 Dec23 ? S 4:56 \_ proftpd: jack - ftpsrv:   IDLE

这样就可以看到proftpd这个进程下面挂了两个线程。
在Linux下面好像因为没有真正的线程，是用进程模拟的，有一个是辅助线程，所以真正程序开的线程应该只有一个。

2. pstree -c也可以达到相同的效果
$ pstree -c | grep proftpd
|-proftpd-+-proftpd
| `-proftpd

3. cat /proc/${pid}/status
可以查看大致的情况

4. pstack
可以查看所有线程的堆栈

