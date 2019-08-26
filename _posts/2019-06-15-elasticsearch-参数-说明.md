---
layout:     post
title:      "elasticsearch参数说明"
date:       2019-05-31 17:03:00
author:     "聼雨夜"
catalog: true
tags:
    - elasticsearch
    - 搜索引擎
---
#### 索引分片分配[Shard allocation]
##### cluster.routing.allocation.enable
```设置集群对未分配(UNASSIGNED)的分片分配方式```
* all 对所有未分配的分片进行分配
* primaries 仅仅对未分配的主分片进行分配
* new_primaries <strong>仅仅对新索引未分配的主分片进行分配(测试没有发现和primaries的不同点)</strong>
* none 所有未分配的主分片都不进行分配

#### 索引分片再平衡[Shard rebalancing]
##### cluster.routing.rebalance.enable
```当集群有新的节点增加或删除时,为了均衡节点间压力,对所有索引分配进行再平衡处理```
* all 对所有类型分片(主分片和副本)都进行rebalance
* primaries 只对主分片进行rebalance
* replicas 只对副本进行rebalance
* none 不对分片进行rebalance,**意味着新增加一个节点,原集群中的分片并不会自动负载均衡到该节点**<br/>
```
如果集群中存储大量的数据,适合将该值设置为none,不然rebalance会造成大量数据往新节点上迁移,最终导致ES集群负载过高,
通过设置为none,然后手动做rebalance,做到ES负载可控,手动rebalance参考resturl:/_cluster/reroute
```    
#### 索引恢复[**Indices Recovery**]

##### 触发条件
* **当集群一个节点丢失,该节点上的分片会被重新分配**
* **elasticsearch集群进行rebalance**
* **手动更改分片分配分配[Shard allocation]**

##### indices.recovery.max_bytes_per_sec
每秒恢复的流量,该值是一个节点(elasticsearch node)级别的限制,包括读和写,也就是说,恢复索引的读和写数据流量
不超过**indices.recovery.max_bytes_per_sec**,可以调整该值来提升索引恢复的速度,如果磁盘压力大(例如:日志系统,恢复的同时,也存在大量的索引任务),
建议降低该值

##### indices.recovery.max_concurrent_file_chunks
恢复索引时,每次恢复并行发送的文件块请求数,如果数据恢复时,实际恢复流量速度没有达到**indices.recovery.max_bytes_per_sec**,
则可以增加该值来提升索引恢复速度

