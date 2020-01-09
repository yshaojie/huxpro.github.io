---
layout:     post
title:      "Elasticsearch批量索引过程源码分析"
date:       2020-01-08 17:03:00
author:     "聼雨夜"
catalog: true
tags:
    - elasticsearch
    - 源码
    - 搜索引擎
---
>Elasticsearch批量索引的业务实现类为**org.elasticsearch.rest.action.document.RestBulkAction**
>最终调用**NodeClient#bulk(BulkRequest, ActionListener<BulkResponse>)** 实现

#### 类继承关系
<img src="https://raw.githubusercontent.com/yshaojie/my-images/master/Elasticsearch/RestIndexActionClass.png"  />

#### 调用流程
**BaseRestHandler#handleRequest(RestRequest request, RestChannel channel, NodeClient client)**-><br/>
**RestBulkAction#prepareRequest(final RestRequest request, final NodeClient client)**-><br/>
**NodeClient.bulk(bulkRequest, new RestStatusToXContentListener<>(channel))**-><br/>

#### 源码解析