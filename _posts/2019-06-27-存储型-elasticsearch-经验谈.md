---
layout:     post
title:      "存储型elasticsearch经验谈"
date:       2019-06-27 17:03:00
author:     "聼雨夜"
catalog: true
tags:
    - elasticsearch
    - ELK
    - 搜索引擎
---
#### Index segment
> indexing buffer 和 index.refresh_interval使得产生segment的间隔变大，从而增加了索引速度

##### indexing buffer
该配置在es启动配置文件elasticsearch.yml里面设置
```text
The indexing buffer is used to store newly indexed documents. When it fills up, the documents in the buffer are written to a segment on disk. It is divided between all shards on the node.

The following settings are static and must be configured on every data node in the cluster:

indices.memory.index_buffer_size
Accepts either a percentage or a byte size value. It defaults to 10%, meaning that 10% of the total heap allocated to a node will be used as the indexing buffer size shared across all shards.
indices.memory.min_index_buffer_size
If the index_buffer_size is specified as a percentage, then this setting can be used to specify an absolute minimum. Defaults to 48mb.
indices.memory.max_index_buffer_size
If the index_buffer_size is specified as a percentage, then this setting can be used to specify an absolute maximum. Defaults to unbounded.
```
```text
If your node is doing only heavy indexing, be sure indices.memory.index_buffer_size is large enough to give at most 512 MB indexing buffer per shard doing heavy indexing (beyond that indexing performance does not typically improve). Elasticsearch takes that setting (a percentage of the java heap or an absolute byte-size), and uses it as a shared buffer across all active shards. Very active shards will naturally use this buffer more than shards that are performing lightweight indexing.
The default is 10% which is often plenty: for example, if you give the JVM 10GB of memory, it will give 1GB to the index buffer, which is enough to host two shards that are heavily indexing.
```

##### index.refresh_interval
该配置在index的settings配置配置项里动态设置
```text
How often to perform a refresh operation, which makes recent changes to the index visible to search. Defaults to 1s
```
增加该配置可以减少segment的产生，但是新数据可索引也相应的增加

##### 建议值
* elasticsearch.yml
```yaml
indices.memory.index_buffer_size=20%
indices.memory.min_index_buffer_size=1GB
indices.memory.max_index_buffer_size=3GB
```
* index 级别设置
```yaml
index.refresh_interval=15s
```

#### Translog
Translog是为了应对elasticsearch节点宕机，索引恢复用的

##### index.translog.sync_interval
多长时间执行一次translog同步到磁盘一次，这个当**index.translog.durability=async**有用

##### index.translog.durability
持久translog到磁盘的方式，默认为**request**,表示每执行一次索引操作便执行translog写磁盘，
**async**表示异步执行translog落盘操作，不过这样会导致没有落盘的translog丢失，最终导致数据的丢失。

##### 建议值
* index级别设置
```yaml
index.translog.sync_interval=30s
index.translog.durability=async
```

#### Shard allocation settings
##### cluster.routing.allocation.same_shard.host
* false(default) - 不进行分片在同一机器上检查
* true - 进行分片在同一机器上检查,保证一个分片(主+副本)不在同一机器上,这样就算机器挂掉,也能保证有可用的分片
该设置是开启检查是否允许索引的一个分片在同一台机器上,默认值为false
**如果存在一个机器上部署了多个实例,建议开启检查,即:true**

#### Shard rebalancing settings
##### cluster.routing.rebalance.enable
* all(default) - 对所有分片进行rebalance
* primaries - 只对主分片进行rebalance
* replicas - 只对副本分片进行rebalance
* none - 不进行rebalance

增加或者删除集群节点时,elasticsearch为了让各个节点上的分片数尽可能的均衡,
默认会进行分片的rebalance,rebalance后,大家的分片数差不多就一样了,这样才能更充分的利用
机器资源,但是存储型elasticsearch集群上边存储了大量的日志,如果进行rebalance,那么rebalance期间会有
大量的数据从硬盘->网络->索引->硬盘走一遍,最终结果就会导致整个集群负载特别高.
这种情况下,适合关闭rebalance,如有需要手动进行rebalance就可以了.
**建议值:none**


#### Shard balancing heuristics
##### cluster.routing.allocation.balance.shard
* 0.45(float) - Defines the weight factor for the total number of shards allocated on a node (float). Defaults to 0.45f. 
                Raising this raises the tendency to equalize the number of shards across all nodes in the cluster

值越大,当有需要创建新的分片时,elasticsearch集群会优先让整个集群的每个节点分片数量更均衡
也就是说,如果集群中有个节点分片数很少,那么创建新分片时,会给该节点多分配分片,来尽快的让整个集群达到
分片数量平衡.<br/>
例如以下场景:<br/>
`cluster.routing.allocation.balance.shard:0.45(默认值)`
`cluster.routing.rebalance.enable:none`,即集群不进行分片的rebalance
现在加入一个新节点,当有新的索引创建时,该索引的所有主分片都会创建在新的节点上,
最终导致新节点负载很高,甚至索引数据产生大量的延迟

#### Total shards per node
##### index.routing.allocation.total_shards_per_node

一个索引在一个集群节点上最大分片分片数量,默认没有限制
调整该值也能避免一个索引太集中于某个节点上,而导致写入或查询导致该节点负载过高


#### Delaying allocation when a node leaves
##### index.unassigned.node_left.delayed_timeout
* 1m(defaul) - 一个分片变为unassigned,如果过了一分钟还处于unassigned状态,则开始对其进行分配

该值主要用于集群中一个节点脱离集群(节点挂了),然后重启,开始恢复它自己的副本
如果节点启动太慢,可以适当的增大`index.unassigned.node_left.delayed_timeout`,保证不会超过delayed_timeout时间
