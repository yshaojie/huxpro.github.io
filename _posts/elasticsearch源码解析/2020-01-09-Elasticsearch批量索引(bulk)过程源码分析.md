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
<img src="https://raw.githubusercontent.com/yshaojie/my-images/master/Elasticsearch/TransportShardBulkActionClass.png"  />

#### 调用流程
**BaseRestHandler#handleRequest(RestRequest request, RestChannel channel, NodeClient client)**-><br/>
**RestBulkAction#prepareRequest(final RestRequest request, final NodeClient client)**-><br/>
**NodeClient.bulk(bulkRequest, new RestStatusToXContentListener<>(channel))**-><br/>

#### 源码解析

##### org.elasticsearch.rest.BaseRestHandler#handleRequest
```java
// prepare the request for execution; has the side effect of touching the request parameters
//调用实现类的prepareRequest方法,
// 这里为org.elasticsearch.rest.action.document.RestBulkAction#prepareRequest
final RestChannelConsumer action = prepareRequest(request, client);

// validate unconsumed params, but we must exclude params used to format the response
// use a sorted set so the unconsumed parameters appear in a reliable sorted order
final SortedSet<String> unconsumedParams =
    request.unconsumedParams().stream().filter(p -> !responseParams().contains(p)).collect(Collectors.toCollection(TreeSet::new));

// validate the non-response params
//存在不能识别的参数,抛出非法的参数异常
if (!unconsumedParams.isEmpty()) {
    final Set<String> candidateParams = new HashSet<>();
    candidateParams.addAll(request.consumedParams());
    candidateParams.addAll(responseParams());
    throw new IllegalArgumentException(unrecognized(request, unconsumedParams, candidateParams, "parameter"));
}

usageCount.increment();
// execute the action
//执行业务逻辑
action.accept(channel);
```

##### org.elasticsearch.rest.action.document.RestBulkAction#prepareRequest
```java
public RestChannelConsumer prepareRequest(final RestRequest request, final NodeClient client) throws IOException {
    BulkRequest bulkRequest = Requests.bulkRequest();
    String defaultIndex = request.param("index");
    String defaultType = request.param("type");
    String defaultRouting = request.param("routing");
    FetchSourceContext defaultFetchSourceContext = FetchSourceContext.parseFromRestRequest(request);
    String fieldsParam = request.param("fields");
    if (fieldsParam != null) {
        DEPRECATION_LOGGER.deprecated("Deprecated field [fields] used, expected [_source] instead");
    }
    String[] defaultFields = fieldsParam != null ? Strings.commaDelimitedListToStringArray(fieldsParam) : null;
    String defaultPipeline = request.param("pipeline");
    String waitForActiveShards = request.param("wait_for_active_shards");
    if (waitForActiveShards != null) {
        bulkRequest.waitForActiveShards(ActiveShardCount.parseString(waitForActiveShards));
    }
    bulkRequest.timeout(request.paramAsTime("timeout", BulkShardRequest.DEFAULT_TIMEOUT));
    bulkRequest.setRefreshPolicy(request.param("refresh"));
    bulkRequest.add(request.requiredContent(), defaultIndex, defaultType, defaultRouting, defaultFields,
        defaultFetchSourceContext, defaultPipeline, null, allowExplicitIndex, request.getXContentType());
    //最终调用client.bulk()来执行索引数据的操作
    return channel -> client.bulk(bulkRequest, new RestStatusToXContentListener<>(channel));
}
```

##### TransportBulkAction#doExecute(Task, BulkRequest, ActionListener<BulkResponse>)
>通过NodeClient的一系列调用,最终调用到**TransportBulkAction#doExecute**,这里正式开始进行数据索引的操作
>调用流程如下

AbstractClient#bulk(BulkRequest, ActionListener<BulkResponse>)<br/>
AbstractClient#execute(Action<Request,Response,RequestBuilder>, Request, ActionListener<Response>)<br/>
NodeClient#doExecute()<br/>
NodeClient#executeLocally(GenericAction<Request,Response>, Request, ActionListener<Response>)<br/>
TransportAction#execute(Request, ActionListener<Response>)<br/>
TransportBulkAction#doExecute(Task, BulkRequest, ActionListener<BulkResponse>)<br/>

```java
protected void doExecute(Task task, BulkRequest bulkRequest, ActionListener<BulkResponse> listener) {
    final long startTime = relativeTime();
    final AtomicArray<BulkItemResponse> responses = new AtomicArray<>(bulkRequest.requests.size());
    //是否需要自动创建索引
    if (needToCheck()) {
        // Attempt to create all the indices that we're going to need during the bulk before we start.
        // Step 1: collect all the indices in the request
        final Set<String> indices = bulkRequest.requests.stream()
                // delete requests should not attempt to create the index (if the index does not
                // exists), unless an external versioning is used
            .filter(request -> request.opType() != DocWriteRequest.OpType.DELETE
                    || request.versionType() == VersionType.EXTERNAL
                    || request.versionType() == VersionType.EXTERNAL_GTE)
            .map(DocWriteRequest::index)
            .collect(Collectors.toSet());
        /* Step 2: filter that to indices that don't exist and we can create. At the same time build a map of indices we can't create
         * that we'll use when we try to run the requests. */
        final Map<String, IndexNotFoundException> indicesThatCannotBeCreated = new HashMap<>();
        //需要自动创建的索引
        Set<String> autoCreateIndices = new HashSet<>();
        ClusterState state = clusterService.state();
        for (String index : indices) {<1>
            boolean shouldAutoCreate;
            try {
                shouldAutoCreate = shouldAutoCreate(index, state);
            } catch (IndexNotFoundException e) {
                shouldAutoCreate = false;
                indicesThatCannotBeCreated.put(index, e);
            }
            if (shouldAutoCreate) {
                autoCreateIndices.add(index);
            }
        }
        // Step 3: create all the indices that are missing, if there are any missing. start the bulk after all the creates come back.
        if (autoCreateIndices.isEmpty()) {
            executeIngestAndBulk(task, bulkRequest, startTime, listener, responses, indicesThatCannotBeCreated);
        } else {
            final AtomicInteger counter = new AtomicInteger(autoCreateIndices.size());
            for (String index : autoCreateIndices) {<2>
                createIndex(index, bulkRequest.timeout(), new ActionListener<CreateIndexResponse>() {
                    @Override
                    public void onResponse(CreateIndexResponse result) {
                        //创建索引的请求任务都完成后,继续执行索引数据操作
                        if (counter.decrementAndGet() == 0) {
                            executeIngestAndBulk(task, bulkRequest, startTime, listener, responses, indicesThatCannotBeCreated);<3>
                        }
                    }

                    @Override
                    public void onFailure(Exception e) {
                        if (!(ExceptionsHelper.unwrapCause(e) instanceof ResourceAlreadyExistsException)) {
                            // fail all requests involving this index, if create didn't work
                            for (int i = 0; i < bulkRequest.requests.size(); i++) {
                                DocWriteRequest request = bulkRequest.requests.get(i);
                                if (request != null && setResponseFailureIfIndexMatches(responses, i, request, index, e)) {
                                    bulkRequest.requests.set(i, null);
                                }
                            }
                        }
                        if (counter.decrementAndGet() == 0) {
                            //创建索引的请求任务都完成后,继续执行索引数据操作
                            executeIngestAndBulk(task, bulkRequest, startTime, ActionListener.wrap(listener::onResponse, inner -> {
                                inner.addSuppressed(e);
                                listener.onFailure(inner);
                            }), responses, indicesThatCannotBeCreated);<3>
                        }
                    }
                });
            }
        }
    } else {
        executeIngestAndBulk(task, bulkRequest, startTime, listener, responses, emptyMap());<3>
    }
}
```
1. 构建需要自动创建的索引
2. for循环发起创建索引请求
3. 请求发送给Ingest节点,继续执行索引数据

##### TransportBulkAction#executeIngestAndBulk
```java
private void executeIngestAndBulk(Task task, final BulkRequest bulkRequest, final long startTimeNanos,
    final ActionListener<BulkResponse> listener, final AtomicArray<BulkItemResponse> responses,
    Map<String, IndexNotFoundException> indicesThatCannotBeCreated) {
    boolean hasIndexRequestsWithPipelines = false;
    final MetaData metaData = clusterService.state().getMetaData();
    ImmutableOpenMap<String, IndexMetaData> indicesMetaData = metaData.indices();
    //遍历所有需要索引的文档,如果有pipeline需求,则hasIndexRequestsWithPipelines = true
    for (DocWriteRequest<?> actionRequest : bulkRequest.requests) {
        IndexRequest indexRequest = getIndexWriteRequest(actionRequest);
        if(indexRequest != null){
            String pipeline = indexRequest.getPipeline();
            if (pipeline == null) {
                IndexMetaData indexMetaData = indicesMetaData.get(actionRequest.index());
                if (indexMetaData == null && indexRequest.index() != null) {
                    //check the alias
                    AliasOrIndex indexOrAlias = metaData.getAliasAndIndexLookup().get(indexRequest.index());
                    if (indexOrAlias != null && indexOrAlias.isAlias()) {
                        AliasOrIndex.Alias alias = (AliasOrIndex.Alias) indexOrAlias;
                        indexMetaData = alias.getWriteIndex();
                    }
                }
                if (indexMetaData == null) {
                    indexRequest.setPipeline(IngestService.NOOP_PIPELINE_NAME);
                } else {
                    String defaultPipeline = IndexSettings.DEFAULT_PIPELINE.get(indexMetaData.getSettings());
                    indexRequest.setPipeline(defaultPipeline);
                    if (IngestService.NOOP_PIPELINE_NAME.equals(defaultPipeline) == false) {
                        hasIndexRequestsWithPipelines = true;
                    }
                }
            } else if (IngestService.NOOP_PIPELINE_NAME.equals(pipeline) == false) {
                hasIndexRequestsWithPipelines = true;
            }
        }
    }
    //需要对待索引文档进行pipeline
    if (hasIndexRequestsWithPipelines) {<1>
        try {
            //当前节点为IngestNode则直接用当前节点进行pipeline操作
            if (clusterService.localNode().isIngestNode()) {
                processBulkIndexIngestRequest(task, bulkRequest, listener);
            } else {
                //将请求转发到Ingest节点
                ingestForwarder.forwardIngestRequest(BulkAction.INSTANCE, bulkRequest, listener);
            }
        } catch (Exception e) {
            listener.onFailure(e);
        }
    } else {<2>
        //没有pipeline需求则直接进行文档索引操作
        executeBulk(task, bulkRequest, startTimeNanos, listener, responses, indicesThatCannotBeCreated);
    }
}
```
1. 有文档需要进行pipeline,如果当前节点为Ingest节点,则当前节点进行pipeline操作,否则随机选一个Ingest节点进行pipeline操作
2. 没有pipeline需求,继续文档索引操作

##### TransportBulkAction.BulkOperation#doRun

```java
protected void doRun() throws Exception {
    final ClusterState clusterState = observer.setAndGetObservedState();
    if (handleBlockExceptions(clusterState)) {
        return;
    }
    final ConcreteIndices concreteIndices = new ConcreteIndices(clusterState, indexNameExpressionResolver);
    MetaData metaData = clusterState.metaData();
    <1> for (int i = 0; i < bulkRequest.requests.size(); i++) {
        //对每个doc进行进一步的处理,如doc id的填充
    }

    // first, go over all the requests and create a ShardId -> Operations mapping
    //创建ShardId->Doc集合的映射,为按分片索引做准备
    Map<ShardId, List<BulkItemRequest>> requestsByShard = new HashMap<>();
    <2> for (int i = 0; i < bulkRequest.requests.size(); i++) {
        DocWriteRequest request = bulkRequest.requests.get(i);
        if (request == null) {
            continue;
        }
        String concreteIndex = concreteIndices.getConcreteIndex(request.index()).getName();
        ShardId shardId = clusterService.operationRouting().indexShards(clusterState, concreteIndex, request.id(),
            request.routing()).shardId();
        List<BulkItemRequest> shardRequests = requestsByShard.computeIfAbsent(shardId, shard -> new ArrayList<>());
        shardRequests.add(new BulkItemRequest(i, request));
    }

    if (requestsByShard.isEmpty()) {
        listener.onResponse(new BulkResponse(responses.toArray(new BulkItemResponse[responses.length()]),
            buildTookInMillis(startTimeNanos)));
        return;
    }

    final AtomicInteger counter = new AtomicInteger(requestsByShard.size());
    String nodeId = clusterService.localNode().getId();
    //按照分片为单位进行doc索引操作
    for (Map.Entry<ShardId, List<BulkItemRequest>> entry : requestsByShard.entrySet()) {
        final ShardId shardId = entry.getKey();
        final List<BulkItemRequest> requests = entry.getValue();
        BulkShardRequest bulkShardRequest = new BulkShardRequest(shardId, bulkRequest.getRefreshPolicy(),
                requests.toArray(new BulkItemRequest[requests.size()]));
        bulkShardRequest.waitForActiveShards(bulkRequest.waitForActiveShards());
        bulkShardRequest.timeout(bulkRequest.timeout());
        if (task != null) {
            bulkShardRequest.setParentTask(nodeId, task.getId());
        }
        <3> shardBulkAction.execute(bulkShardRequest, new ActionListener<BulkShardResponse>() {
            @Override
            public void onResponse(BulkShardResponse bulkShardResponse) {
                for (BulkItemResponse bulkItemResponse : bulkShardResponse.getResponses()) {
                    // we may have no response if item failed
                    if (bulkItemResponse.getResponse() != null) {
                        bulkItemResponse.getResponse().setShardInfo(bulkShardResponse.getShardInfo());
                    }
                    responses.set(bulkItemResponse.getItemId(), bulkItemResponse);
                }
                //所有分片请求都处理完成则返回整个请求结果
                if (counter.decrementAndGet() == 0) {
                    finishHim();
                }
            }

            @Override
            public void onFailure(Exception e) {
                // create failures for all relevant requests
                for (BulkItemRequest request : requests) {
                    final String indexName = concreteIndices.getConcreteIndex(request.index()).getName();
                    DocWriteRequest docWriteRequest = request.request();
                    responses.set(request.id(), new BulkItemResponse(request.id(), docWriteRequest.opType(),
                            new BulkItemResponse.Failure(indexName, docWriteRequest.type(), docWriteRequest.id(), e)));
                }
                //所有分片请求都处理完成则返回整个请求结果
                if (counter.decrementAndGet() == 0) {
                    finishHim();
                }
            }

            private void finishHim() {
                listener.onResponse(new BulkResponse(responses.toArray(new BulkItemResponse[responses.length()]),
                    buildTookInMillis(startTimeNanos)));
            }
        });
    }
}
```
1. 对每个doc进行进一步的处理,如doc id的填充
2. 创建ShardId->Doc集合的映射,为按分片索引做准备
3. 按照分片为单位进行分布式索引操作,至此进入TransportShardBulkAction.execute()

##### 索引过程
**TransportShardBulkAction.execute**通过层层的调用,最终调用到方法**TransportReplicationAction#doExecute(Task, Request, ActionListener)**
该方法通过运行**new ReroutePhase((ReplicationTask) task, request, listener).run()** 开启了 **routing**阶段.

* org.elasticsearch.action.support.replication.TransportReplicationAction.ReroutePhase#doRun
```java
protected void doRun() {
    //设置为routing阶段
    setPhase(task, "routing");
    final ClusterState state = observer.setAndGetObservedState();
    final String concreteIndex = concreteIndex(state, request);
    final ClusterBlockException blockException = blockExceptions(state, concreteIndex);
    if (blockException != null) {
        if (blockException.retryable()) {
            logger.trace("cluster is blocked, scheduling a retry", blockException);
            //可以重试的话,则等待集群状态变化后进行重试
            retry(blockException);
        } else {
            finishAsFailed(blockException);
        }
    } else {
        // request does not have a shardId yet, we need to pass the concrete index to resolve shardId
        final IndexMetaData indexMetaData = state.metaData().index(concreteIndex);
        if (indexMetaData == null) {
            retry(new IndexNotFoundException(concreteIndex));
            return;
        }
        if (indexMetaData.getState() == IndexMetaData.State.CLOSE) {
            throw new IndexClosedException(indexMetaData.getIndex());
        }

        // resolve all derived request fields, so we can route and apply it
        resolveRequest(indexMetaData, request);
        assert request.shardId() != null : "request shardId must be set in resolveRequest";
        assert request.waitForActiveShards() != ActiveShardCount.DEFAULT :
            "request waitForActiveShards must be set in resolveRequest";

        final ShardRouting primary = primary(state);
        if (retryIfUnavailable(state, primary)) {
            return;
        }
        final DiscoveryNode node = state.nodes().get(primary.currentNodeId());
        if (primary.currentNodeId().equals(state.nodes().getLocalNodeId())) {
            //如果是主分片在当前节点,则直接在当前节点执行请求
            performLocalAction(state, primary, node, indexMetaData);
        } else {
            //将请求转发到主分片所在节点
            performRemoteAction(state, primary, node);
        }
    }
}
```
**routing**主要就是找到主分片所在的节点然后把任务转发过去,如果当前节点正式该主分片所在节点,
那么直接**TransportReplicationAction.ReroutePhase#performLocalAction**,然后进入**waiting_on_primary**阶段.

**performLocalAction**最终调用**TransportService#sendRequest**执行bulk doc操作,
**TransportService#sendRequest**分两步
1. Transport.Connection connection = getConnection(node); 第一步来获取一个连接,如果node为LocalNode,
返回TransportService#localNodeConnection,否则返回Remote Socket连接.
2. 用第一步返回的连接执行请求
从边我们得知,最终用**TransportService#localNodeConnection**来执行本地bulk index操作,
其是一个内部实现,源码如下
```java
private final Transport.Connection localNodeConnection = new Transport.Connection() {
    @Override
    public DiscoveryNode getNode() {
        return localNode;
    }

    @Override
    public void sendRequest(long requestId, String action, TransportRequest request, TransportRequestOptions options)
        throws TransportException {
        //此处为执行本地索引操作的入口
        <1>sendLocalRequest(requestId, action, request, options);
    }

    @Override
    public void addCloseListener(ActionListener<Void> listener) {
    }

    @Override
    public boolean isClosed() {
        return false;
    }

    @Override
    public void close() {
    }
};
```
1. TransportService#sendLocalRequest为真正执行bulk index的入口
```java
private void sendLocalRequest(long requestId, final String action, final TransportRequest request, TransportRequestOptions options) {
    final DirectResponseChannel channel = new DirectResponseChannel(logger, localNode, action, requestId, this, threadPool);
    try {
        onRequestSent(localNode, requestId, action, request, options);
        onRequestReceived(requestId, action);
        //获取该action对应的RequestHandlerRegistry
        //这些handler都是通过方法TransportService.registerRequestHandler()注册进来的
        <1>final RequestHandlerRegistry reg = getRequestHandler(action);
        if (reg == null) {
            throw new ActionNotFoundTransportException("Action [" + action + "] not found");
        }
        final String executor = reg.getExecutor();
        if (ThreadPool.Names.SAME.equals(executor)) {
            //noinspection unchecked
            //执行请求
            <2> reg.processMessageReceived(request, channel);
        } else {
            threadPool.executor(executor).execute(new AbstractRunnable() {
                @Override
                protected void doRun() throws Exception {
                    //noinspection unchecked
                    //执行请求
                    <2> reg.processMessageReceived(request, channel);
                }

                
            });
        }
        //do something
    } catch (Exception e) {
        //do something
    }
```
**TransportService#sendLocalRequest**主要做了两件事情
1. 获取action对应的RequestHandlerRegistry,类**TransportReplicationAction**在构造方法里面注册了3个RequestHandlerRegistry,分别为
    * indices:data/write/bulk[s] 没确定
    * indices:data/write/bulk[s][p] 主分片的写
    * indices:data/write/bulk[s][r] 副本的写
2. 调用RequestHandlerRegistry#processMessageReceived(request, channel)


#### 待解
* ClusterBlockException
* Index Phase 