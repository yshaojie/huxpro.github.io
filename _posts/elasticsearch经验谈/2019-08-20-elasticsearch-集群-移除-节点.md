---
layout:     post
title:      "elasticsearch移除数据节点"
date:       2019-08-20 17:03:00
author:     "聼雨夜"
catalog: true
tags:
    - elasticsearch
    - 搜索引擎
---
### 概述
>有时我们需要从elasticsearch集群中移除一个数据节点,那么就涉及到数据迁移<br/>
>elasticsearch给我们提供了一个集群级别的参数来达到此目的

### 操作流程
#### 排除节点
>引用原文:<br/>
>The most common use case for cluster-level shard allocation filtering is when you want to decommission a node. To move shards off of a node prior to shutting it down, you could create a filter that excludes the node by its IP address:

```bash
curl -X PUT "localhost:9200/_cluster/settings?pretty" -H 'Content-Type: application/json' -d'
{
  "transient" : {
    "cluster.routing.allocation.exclude._name" : "es name",
    "cluster.routing.allocation.exclude._ip" : "10.0.0.1",
    "cluster.routing.allocation.exclude._host" : "host"
  }
}
'
```
* 支持三种(**_name,_ip,_host**)来排除要移除的节点,支持通配符,这样新建的索引分片不会在<br/>
  该节点上创建,并且该节点的分片也会转移到其他节点上,直到迁移完毕
#### 移除节点
1. 等该节点所有的分片都迁移完毕后,就可以将该集群节点关闭
2. 将集群的*cluster.routing.allocation.*参数恢复,如下
```bash
curl -X PUT "localhost:9200/_cluster/settings?pretty" -H 'Content-Type: application/json' -d'
{
  "transient" : {
    "cluster.routing.allocation.exclude._name" : null,
    "cluster.routing.allocation.exclude._ip" : null,
    "cluster.routing.allocation.exclude._host" : null
  }
}
'
```