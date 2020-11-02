---
layout:     post
title:      "netty 配置 汇总"
date:       2019-03-22 12:00:00
author:     "聼雨夜"
catalog: true
tags:
    - linux
    - io
    - buffer/cache
    - disk
---
#### buffer/cache概念
通过free命令可以查看系统的buffer/cache占用情况,如图
![Public License](/img/in-post/linux-command-freem.png)
图中显示当前系统的buffer/cache一共占用2897Mb
那buffer和cache到底代表什么含义,我们可以通过man free来查看他们的说明
![Public License](/img/in-post/man-free-desc.png)
从文档的描述来看,buffer和cache的真实来源是在/proc/meminfo里面,那么继续深入来看,执行 man proc
![Public License](/img/in-post/linux-man-proc.png)
从proc我们可以看出buffer和cache的具体定义了
buffer是针对磁盘的缓冲
cache是针对文件的缓存
需要说明的是,我们程序一般都是通过文件来进行读写,所以大多时候我们关注cache即可,通过命令 free -w即可
区分出buffer和cache具体占用内存多少
#### cache对文件读写的影响

##### SO_KEEPALIVE

- 参考 [TCP持久连接](https://en.wikipedia.org/wiki/Keepalive#TCP_keepalive "TCP持久连接")

##### SO_TIMEOUT

- 参考[socket time out 理解](https://cloud.tencent.com/developer/article/1039881 "socket time out 理解")
