---
layout:     post
title:      "Elasticsearch的rest请求处理流程"
date:       2020-01-03 17:03:00
author:     "聼雨夜"
catalog: true
tags:
    - elasticsearch
    - 源码
    - 搜索引擎
---
>本文章以netty为实现来讲解下elasticsearch的rest处理流程
#### 处理Rest Request的netty Handler
处理Elasticsearch Rest Request的业务Handler为**org.elasticsearch.http.netty4.Netty4HttpRequestHandler**

##### org.elasticsearch.http.netty4.Netty4HttpRequestHandler#channelRead0
```java

```
```flow js
st=>start: Start
op=>operation: Your Operation
cond=>condition: Yes or No?
e=>end

st->op->cond
cond(yes)->e
cond(no)->op
```


