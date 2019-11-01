---
title: TIME_WAIT和CLOSE_WAIT
link_title: timeWaitCloseWait
date: 2017-07-13 10:48:05
tags: [网络]
categories: 网络
thumbnailImage: https://i.loli.net/2019/09/24/UtQusjEKVgboBHP.png	
thumbnailImagePosition: left
---
<span/>
<!-- more -->
![](https://i.loli.net/2019/09/24/UtQusjEKVgboBHP.png)
<!-- toc -->
# TIME_WAIT和CLOSE_WAIT

在服务器的日常维护过程中，会经常用到下面的命令：

> netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'    

它会显示例如下面的信息：
> TIME_WAIT 814
  CLOSE_WAIT 1
  FIN_WAIT1 1
  ESTABLISHED 634
  SYN_RECV 2
  LAST_ACK 1

常用的三个状态是：
- ESTABLISHED 表示正在通信
- TIME_WAIT 表示主动关闭
- CLOSE_WAIT 表示被动关闭。

这几种状态什么意思呢，看看图

![](timeWaitcloseWait/01.png)

tcp断开时候也可能是图中的情况，反过来所以说服务端可能处于TIME_WAIT状态，也有可能处于CLOSE_WAIT状态


一般不到万不得已的情况也不会去查看网络状态，如果服务器出了异常，百分之八九十都是下面两种情况：
1. 服务器保持了大量TIME_WAIT状态
2. 服务器保持了大量CLOSE_WAIT状态

因为Linux分配给一个用户的文件句柄是有限的，而TIME_WAIT和CLOSE_WAIT两种状态如果一直被保持，那么意味着对应数目的通道就一直被占着，而且是“占着茅坑不使劲”，一旦达到句柄数上限，新的请求就无法被处理了，接着就是大量Too Many Open Files异常，tomcat崩溃


# 如何解决存在大量TIME_WAIT和CLOSE_WAIT的问题
## 减少TIME_WAIT状态
解决思路很简单，就是让服务器能够快速回收和重用那些TIME_WAIT的资源。
对**/etc/sysctl.conf**文件修改
```xml
#对于一个新建连接，内核要发送多少个 SYN 连接请求才决定放弃,不应该大于255，默认值是5，对应于180秒左右时间   
net.ipv4.tcp_syn_retries=2  
#net.ipv4.tcp_synack_retries=2  
#表示当keepalive起用的时候，TCP发送keepalive消息的频度。缺省是2小时，改为300秒  
net.ipv4.tcp_keepalive_time=1200  
net.ipv4.tcp_orphan_retries=3  
#表示如果套接字由本端要求关闭，这个参数决定了它保持在FIN-WAIT-2状态的时间  
net.ipv4.tcp_fin_timeout=30    
#表示SYN队列的长度，默认为1024，加大队列长度为8192，可以容纳更多等待连接的网络连接数。  
net.ipv4.tcp_max_syn_backlog = 4096  
#表示开启SYN Cookies。当出现SYN等待队列溢出时，启用cookies来处理，可防范少量SYN攻击，默认为0，表示关闭  
net.ipv4.tcp_syncookies = 1  
  
#表示开启重用。允许将TIME-WAIT sockets重新用于新的TCP连接，默认为0，表示关闭  
net.ipv4.tcp_tw_reuse = 1  
#表示开启TCP连接中TIME-WAIT sockets的快速回收，默认为0，表示关闭  
net.ipv4.tcp_tw_recycle = 1  
  
##减少超时前的探测次数   
net.ipv4.tcp_keepalive_probes=5   
##优化网络设备接收队列   
net.core.netdev_max_backlog=3000   
```
修改完之后执行**/sbin/sysctl -p**让参数生效。

要注意的几个参数：
- net.ipv4.tcp_tw_reuse
- net.ipv4.tcp_tw_recycle
- net.ipv4.tcp_fin_timeout 
- net.ipv4.tcp_keepalive_*

net.ipv4.tcp_tw_reuse和net.ipv4.tcp_tw_recycle的开启都是为了回收处于TIME_WAIT状态的资源。

net.ipv4.tcp_fin_timeout这个时间可以减少在异常情况下服务器从FIN-WAIT-2转到TIME_WAIT的时间

net.ipv4.tcp_keepalive_*一系列参数，是用来设置服务器检测连接存活的相关配置


# 减少CLOSE_WAIT状态
TIME_WAIT状态可以通过优化服务器参数得到解决，因为发生TIME_WAIT的情况是服务器自己可控的，要么就是对方连接的异常，要么就是自己没有迅速回收资源，总之不是由于自己程序错误导致的。

但是CLOSE_WAIT就不一样了，从上面的图可以看出来，如果一直保持在CLOSE_WAIT状态，**那么只有一种情况，就是在对方关闭连接之后服务器程序自己没有进一步发出ack信号。**换句话说，就是在对方连接关闭之后，程序里没有检测到，或者程序压根就忘记了这个时候需要关闭连接，于是这个资源就一直被程序占着。

个人觉得这种情况，通过服务器内核参数也没办法解决，服务器对于程序抢占的资源没有主动回收的权利，除非终止程序运行。

所以如果将大量CLOSE_WAIT的解决办法总结为一句话那就是：查代码。因为问题出在服务器程序里头啊。

参考:
- http://blog.csdn.net/shootyou/article/details/6622226

