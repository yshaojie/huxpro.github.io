---
layout:     post
title:      "buffer cache理解"
date:       2020-11-02 12:00:00
author:     "聼雨夜"
catalog: true
tags:
    - linux
    - io
    - buffer/cache
    - disk
---
#### buffer/cache概念
通过free命令可以查看系统的buffer/cache占用情况,如下
```text
root@Think:~# free -m
              total        used        free      shared  buff/cache   available
Mem:          13937        6071        3996         429        3869        7117
Swap:          2047           0        2047
```
图中显示当前系统的buffer/cache一共占用3869Mb
那buffer和cache到底代表什么含义,我们可以通过man free来查看他们的说明
```text
buffers
      Memory used by kernel buffers (Buffers in /proc/meminfo)

cache  Memory used by the page cache and slabs (Cached and SReclaimable in /proc/meminfo)

buff/cache
      Sum of buffers and cache
```
从文档的描述来看,buffer和cache的真实来源是在/proc/meminfo里面,那么继续深入来看,执行 man proc
```text
Buffers %lu
     Relatively temporary storage for raw disk blocks that shouldn't get tremendously large (20MB or so).

Cached %lu
     In-memory cache for files read from the disk (the page cache).  Doesn't include SwapCached.
```
从proc我们可以看出buffer和cache的具体定义了<br/>
*buffer是针对磁盘的缓冲
*cache是针对文件的缓存
需要说明的是,我们程序一般都是通过文件来进行读写,所以大多时候我们关注cache即可,<br/>
通过命令 **free -w**即可区分出buffer和cache具体占用内存多少

#### cache对文件读写的影响
通过命令**echo 3 > /proc/sys/vm/drop_caches**可以清除buffer/cache

##### buffer/cache对写文件的影响
写入一个1GB的文件<br/>
写入前<
```text
root@Think:~# free -mw
              total        used        free      shared     buffers       cache   available
Mem:          13937       10447        1890         793          17        1582        2422
Swap:          2047         340        1707
root@Think:~# 
```
写入后
```text
root@Think:~# free -mw
              total        used        free      shared     buffers       cache   available
Mem:          13937       10472         750         792          28        2687        2385
Swap:          2047         340        1707
root@Think:~# 
```
这里我们可以看出,普通文件的写也用到了cache,会把新写入的文件缓存起来<br/>

##### buffer/cache对读文件的影响
紧接上边写文件后,直接开始读写入的文件<br/>
```text
#iostat -x nvme0n1 -k -d 3部分打印结果
Device            r/s     w/s     rkB/s     wkB/s   rrqm/s   wrqm/s  %rrqm  %wrqm r_await w_await aqu-sz rareq-sz wareq-sz  svctm  %util
nvme0n1          1.00    0.33     44.00      1.33     0.00     0.00   0.00   0.00    1.33    1.00   0.00    44.00     4.00   3.00   0.40

Device            r/s     w/s     rkB/s     wkB/s   rrqm/s   wrqm/s  %rrqm  %wrqm r_await w_await aqu-sz rareq-sz wareq-sz  svctm  %util
nvme0n1          0.00    4.33      0.00    377.33     0.00    13.67   0.00  75.93    0.00    0.92   0.00     0.00    87.08   2.15   0.93

Device            r/s     w/s     rkB/s     wkB/s   rrqm/s   wrqm/s  %rrqm  %wrqm r_await w_await aqu-sz rareq-sz wareq-sz  svctm  %util
nvme0n1          1.67    5.33     46.67     74.67    10.00     3.00  85.71  36.00    1.40    0.38   0.00    28.00    14.00   1.71   1.20

Device            r/s     w/s     rkB/s     wkB/s   rrqm/s   wrqm/s  %rrqm  %wrqm r_await w_await aqu-sz rareq-sz wareq-sz  svctm  %util
nvme0n1          0.33    0.67      1.33     58.67     0.00    14.00   0.00  95.45    6.00    3.50   0.00     4.00    88.00   5.33   0.53

Device            r/s     w/s     rkB/s     wkB/s   rrqm/s   wrqm/s  %rrqm  %wrqm r_await w_await aqu-sz rareq-sz wareq-sz  svctm  %util
nvme0n1          0.00    1.00      0.00     13.33     0.00     2.33   0.00  70.00    0.00    2.33   0.00     0.00    13.33   4.00   0.40

Device            r/s     w/s     rkB/s     wkB/s   rrqm/s   wrqm/s  %rrqm  %wrqm r_await w_await aqu-sz rareq-sz wareq-sz  svctm  %util
nvme0n1          1.00    6.33     44.00    448.00    10.00    19.67  90.91  75.64    2.67    0.79   0.00    44.00    70.74   1.82   1.33
```
从r/s列我们可以看出,磁盘读的qps很低,说明本次读我们并没有直接读磁盘,而是读取的cache里面数据<br/>
清除cache后继续再读,执行命令**echo 3 > /proc/sys/vm/drop_caches**<br/>
清除后的buffer/cache<br/>
```text
root@Think:~# free -m
              total        used        free      shared  buff/cache   available
Mem:          13937        6421        5948         510        1568        6739
Swap:          2047           0        2047
```

```text
#iostat -x nvme0n1 -k -d 3部分打印结果
Device            r/s     w/s     rkB/s     wkB/s   rrqm/s   wrqm/s  %rrqm  %wrqm r_await w_await aqu-sz rareq-sz wareq-sz  svctm  %util
nvme0n1        162.33    8.33  20696.00     64.00     0.00     1.33   0.00  13.79    0.09    0.08   0.00   127.49     7.68   3.85  65.73

Device            r/s     w/s     rkB/s     wkB/s   rrqm/s   wrqm/s  %rrqm  %wrqm r_await w_await aqu-sz rareq-sz wareq-sz  svctm  %util
nvme0n1        164.67    0.67  20993.33     26.67     0.00     6.00   0.00  90.00    0.09    0.50   0.00   127.49    40.00   4.06  67.07

Device            r/s     w/s     rkB/s     wkB/s   rrqm/s   wrqm/s  %rrqm  %wrqm r_await w_await aqu-sz rareq-sz wareq-sz  svctm  %util
nvme0n1        163.33    0.67  20698.67     37.33     0.00     0.00   0.00   0.00    0.09    0.00   0.00   126.73    56.00   4.01  65.73

Device            r/s     w/s     rkB/s     wkB/s   rrqm/s   wrqm/s  %rrqm  %wrqm r_await w_await aqu-sz rareq-sz wareq-sz  svctm  %util
nvme0n1        163.00   48.67  20693.33    644.00     0.00    26.00   0.00  34.82    0.11    0.09   0.00   126.95    13.23   3.14  66.40

Device            r/s     w/s     rkB/s     wkB/s   rrqm/s   wrqm/s  %rrqm  %wrqm r_await w_await aqu-sz rareq-sz wareq-sz  svctm  %util
nvme0n1        138.67    0.33  17624.00      1.33     0.00     0.00   0.00   0.00    0.09    1.00   0.00   127.10     4.00   4.09  56.80

Device            r/s     w/s     rkB/s     wkB/s   rrqm/s   wrqm/s  %rrqm  %wrqm r_await w_await aqu-sz rareq-sz wareq-sz  svctm  %util
nvme0n1          1.00   16.00      4.00   2360.00     0.00    23.33   0.00  59.32    0.67    0.46   0.00     4.00   147.50   0.78   1.33

Device            r/s     w/s     rkB/s     wkB/s   rrqm/s   wrqm/s  %rrqm  %wrqm r_await w_await aqu-sz rareq-sz wareq-sz  svctm  %util
nvme0n1          0.00    0.33      0.00     16.00     0.00     0.33   0.00  50.00    0.00    4.00   0.00     0.00    48.00   8.00   0.27

#读取完毕后的buffer/cache结果
root@Think:~# free -m
              total        used        free      shared  buff/cache   available
Mem:          13937        6525        4674         534        2738        6611
Swap:          2047           0        2047
```
从这里我们看出<br/>
1. 执行了**echo 3 > /proc/sys/vm/drop_caches**之后再次读文件,磁盘的r/s明显读的磁盘,当文件读取完毕,r/s又降了下去,
    说明本次读的是磁盘
2. 通过读取前后**free -m**的对比,cache明显增大1G+,说明 我们本次读取文件,把文件内容也缓存了起来

结论:
1. **程序对磁盘的读写都会将文件内容进行cache**
2. **读的时候优先读cache内容然后再读磁盘**

#### buffer/cache的大小
> 接下来我们测试系统参数对buffer/cache的影响

不做任何调整的情况下,进行6次1GB内容的写入
```text
#写之前
root@Think:~# free -m
              total        used        free      shared  buff/cache   available
Mem:          13937        7056        5466         391        1414        6219
Swap:          2047           0        2047
#第一次写入后
root@Think:~# free -m
              total        used        free      shared  buff/cache   available
Mem:          13937        7071        4330         396        2536        6187
Swap:          2047           0        2047
#第二次写入后
root@Think:~# free -m
              total        used        free      shared  buff/cache   available
Mem:          13937        7073        3260         395        3604        6171
Swap:          2047           0        2047
#第三次写入后
root@Think:~# free -m
              total        used        free      shared  buff/cache   available
Mem:          13937        7062        3258         391        3616        6187
Swap:          2047           0        2047
#第四次写入后
root@Think:~# free -m
              total        used        free      shared  buff/cache   available
Mem:          13937        7061        3247         391        3628        6187
Swap:          2047           0        2047
#第五次写入后
root@Think:~# free -m
              total        used        free      shared  buff/cache   available
Mem:          13937        7063        3235         391        3639        6186
Swap:          2047           0        2047

```
从结果看,第三次写入后buffer/cache就没再增长,而此时我的内存还存在3GB+<br/>
说明buffer/cache不会无限的吃内存.<br/>
通过查看网络资料,跟buffer/cache相关的配置如下
* vm.vfs_cache_pressure
```text
This option controls the tendency of the kernel to reclaim 
the memory which is used for caching of directory and inode objects.
At the default value of vfs_cache_pressure=100 the kernel will attempt to reclaim dentries and 
inodes at a "fair" rate with respect to pagecache and swapcache reclaim. Decreasing vfs_cache_pressure 
causes the kernel to prefer to retain dentry and inode caches. When vfs_cache_pressure=0, 
the kernel will never reclaim dentries and inodes due to memory pressure and this can easily lead to out-of-memory conditions. 
Increasing vfs_cache_pressure beyond 100 causes the kernel to prefer to reclaim dentries and inodes.
```
**vm.vfs_cache_pressure来控制回收buffer/cache的程度,默认100,越大越利于buffer/cache的回收,
坏处就是磁盘不能够充分的利用内存来提高性能**

* cgroups
东西比较多,还没研究过

#### 总结
* 对磁盘的读写都会利用page cache来提高磁盘的读写性能
* buffer/cache不能通过参数直接设置,其大小是操作系统根据目前内存使用的繁忙程度动态计算的,
  操作系统本身还是以应用使用内存为优先的,当系统内存比较吃紧的时候,buffer/cache可用的空间就较少.
* buffer/cache不会无限吃掉内存,当buffer/cache达到上限时,新的磁盘读写时,操作系统会清除没有正在使用的cache,
  从而让新的读写内容进入cache.
* 对于小文件(buffer/cache上限以下)可以提高读写的性能
* 由于buffer/cache的大小不确定性,所以这点对磁盘性能来说存在很多变数
* 如果想重复利用buffer/cache,就要确保操作系统有足够多的空闲内存

#### 参考
- <https://tldp.org/LDP/sag/html/buffer-cache.html>
- <https://www.kernel.org/doc/Documentation/sysctl/vm.txt>
- <https://stackoverflow.com/questions/47412846/how-to-find-the-size-of-buffer-cache-used-in-file-system-of-linux>

