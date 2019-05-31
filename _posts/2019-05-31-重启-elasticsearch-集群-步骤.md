---
layout:     post
title:      "重启elasticsearch集群步骤"
date:       2019-05-31 17:03:00
author:     "聼雨夜"
catalog: true
tags:
    - elasticsearch
    - 搜索引擎
---
### 说明
* 该方案是基于elasticsearch 6.x,其它版本是否合适请自行评估
* 该方案是基于滚动升级elasticsearch里面的关闭和启动elasticsearch步骤
### 步骤
#### 1.Disable shard allocation
```
curl -X PUT "localhost:9200/_cluster/settings" -H 'Content-Type: application/json' -d'
{
  "persistent": {
    "cluster.routing.allocation.enable": "none"
  }
}
'
```
#### 2.Stop indexing and perform a synced flush.
```
curl -X POST "localhost:9200/_flush/synced"
```
**When you perform a synced flush, check the response to make sure there are no failures. Synced flush operations that fail due to pending indexing operations are listed in the response body, although the request itself still returns a 200 OK status. If there are failures, reissue the request**

#### 3.Stop any machine learning jobs that are running

#### 4.Restart Elasticsearch Node 

#### 5.Reenable allocation.
```
curl -X PUT "localhost:9200/_cluster/settings" -H 'Content-Type: application/json' -d'
{
  "persistent": {
    "cluster.routing.allocation.enable": null
  }
}
'
```
#### 6.持续观察重启的节点,数据恢复情况,直到集群为green状态
```
curl -X GET "localhost:9200/_cat/health?v"
curl -X GET "localhost:9200/_cat/recovery"
```
#### 7.Restart machine learning jobs.

