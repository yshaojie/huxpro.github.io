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

#### Netty4HttpServerTransport#doStart
rest服务启动入口,doStart方法我们看到了熟悉的netty代码<br/>
这些代码用来初始化和启动rest服务
```java
protected void doStart() {
        boolean success = false;
        try {
            this.serverOpenChannels = new Netty4OpenChannelsHandler(logger);
            //do something
            serverBootstrap.childHandler(configureServerChannelHandler());<1>
            //do something
            this.boundAddress = createBoundHttpAddress();<2>
            if (logger.isInfoEnabled()) {
                logger.info("{}", boundAddress);
            }
            success = true;
        } finally {
            if (success == false) {
                doStop(); // otherwise we leak threads since we never moved to started
            }
        }
    }
```
1. 初始化netty编解码流程,最终逻辑在**org.elasticsearch.http.netty4.Netty4HttpServerTransport.HttpChannelHandler#initChannel**里面
2. bind socket,接收请求

#### org.elasticsearch.http.netty4.Netty4HttpServerTransport.HttpChannelHandler#initChannel
```java
protected void initChannel(Channel ch) throws Exception {
            ch.pipeline().addLast("openChannels", transport.serverOpenChannels);
            ch.pipeline().addLast("read_timeout", new ReadTimeoutHandler(transport.readTimeoutMillis, TimeUnit.MILLISECONDS));
            final HttpRequestDecoder decoder = new HttpRequestDecoder(
                Math.toIntExact(transport.maxInitialLineLength.getBytes()),
                Math.toIntExact(transport.maxHeaderSize.getBytes()),
                Math.toIntExact(transport.maxChunkSize.getBytes()));
            decoder.setCumulator(ByteToMessageDecoder.COMPOSITE_CUMULATOR);
            ch.pipeline().addLast("decoder", decoder);
            ch.pipeline().addLast("decoder_compress", new HttpContentDecompressor());
            ch.pipeline().addLast("encoder", new HttpResponseEncoder());
            final HttpObjectAggregator aggregator = new HttpObjectAggregator(Math.toIntExact(transport.maxContentLength.getBytes()));
            aggregator.setMaxCumulationBufferComponents(transport.maxCompositeBufferComponents);
            ch.pipeline().addLast("aggregator", aggregator);
            if (transport.compression) {
                ch.pipeline().addLast("encoder_compress", new HttpContentCompressor(transport.compressionLevel));
            }
            if (SETTING_CORS_ENABLED.get(transport.settings())) {
                ch.pipeline().addLast("cors", new Netty4CorsHandler(transport.getCorsConfig()));
            }
            if (transport.pipelining) {<1>
                ch.pipeline().addLast("pipelining", new HttpPipeliningHandler(logger, transport.pipeliningMaxEvents));
            }
            ch.pipeline().addLast("handler", requestHandler);<2>
        }
```
1. 支持HTTP pipelining
2. 真正处理业务请求的Handler,实现为**org.elasticsearch.http.netty4.Netty4HttpRequestHandler**

##### org.elasticsearch.http.netty4.Netty4HttpRequestHandler#channelRead0
```java
protected void channelRead0(ChannelHandlerContext ctx, Object msg) throws Exception {
    final FullHttpRequest request;
    final HttpPipelinedRequest pipelinedRequest;
    if (this.httpPipeliningEnabled && msg instanceof HttpPipelinedRequest) {
        pipelinedRequest = (HttpPipelinedRequest) msg;
        request = (FullHttpRequest) pipelinedRequest.last();
    } else {
        pipelinedRequest = null;
        request = (FullHttpRequest) msg;
    }

    boolean success = false;
    try {

        final FullHttpRequest copy =
                new DefaultFullHttpRequest(
                        request.protocolVersion(),
                        request.method(),
                        request.uri(),
                        Unpooled.copiedBuffer(request.content()),
                        request.headers(),
                        request.trailingHeaders());

        Exception badRequestCause = null;

        /*
         * We want to create a REST request from the incoming request from Netty. However, creating this request could fail if there
         * are incorrectly encoded parameters, or the Content-Type header is invalid. If one of these specific failures occurs, we
         * attempt to create a REST request again without the input that caused the exception (e.g., we remove the Content-Type header,
         * or skip decoding the parameters). Once we have a request in hand, we then dispatch the request as a bad request with the
         * underlying exception that caused us to treat the request as bad.
         */
        final Netty4HttpRequest httpRequest;
        {
            Netty4HttpRequest innerHttpRequest;
            try {
                innerHttpRequest = new Netty4HttpRequest(serverTransport.xContentRegistry, copy, ctx.channel());
            } catch (final RestRequest.ContentTypeHeaderException e) {
                badRequestCause = e;
                innerHttpRequest = requestWithoutContentTypeHeader(copy, ctx.channel(), badRequestCause);
            } catch (final RestRequest.BadParameterException e) {
                badRequestCause = e;
                innerHttpRequest = requestWithoutParameters(copy, ctx.channel());
            }
            httpRequest = innerHttpRequest;
        }


        /*
         * We now want to create a channel used to send the response on. However, creating this channel can fail if there are invalid
         * parameter values for any of the filter_path, human, or pretty parameters. We detect these specific failures via an
         * IllegalArgumentException from the channel constructor and then attempt to create a new channel that bypasses parsing of these
         * parameter values.
         */
        final Netty4HttpChannel channel;
        {
            Netty4HttpChannel innerChannel;
            try {
                innerChannel =
                        new Netty4HttpChannel(serverTransport, httpRequest, pipelinedRequest, detailedErrorsEnabled, threadContext);
            } catch (final IllegalArgumentException e) {
                if (badRequestCause == null) {
                    badRequestCause = e;
                } else {
                    badRequestCause.addSuppressed(e);
                }
                final Netty4HttpRequest innerRequest =
                        new Netty4HttpRequest(
                                serverTransport.xContentRegistry,
                                Collections.emptyMap(), // we are going to dispatch the request as a bad request, drop all parameters
                                copy.uri(),
                                copy,
                                ctx.channel());
                innerChannel =
                        new Netty4HttpChannel(serverTransport, innerRequest, pipelinedRequest, detailedErrorsEnabled, threadContext);
            }
            channel = innerChannel;
        }

        if (request.decoderResult().isFailure()) {
            serverTransport.dispatchBadRequest(httpRequest, channel, request.decoderResult().cause());
        } else if (badRequestCause != null) {
            serverTransport.dispatchBadRequest(httpRequest, channel, badRequestCause);
        } else {
            serverTransport.dispatchRequest(httpRequest, channel);<1>
        }
        success = true;
    } finally {
        // the request is otherwise released in case of dispatch
        if (success == false && pipelinedRequest != null) {
            pipelinedRequest.release();
        }
    }
}
```
1. 封装RestRequest和RestChannel,通过Netty4HttpServerTransport#dispatchRequest进去请求分发处理
    其中RestRequest包含了http请求信息,Channel也也包含了请求信息和持有Netty channel对象,个人觉的这块有点乱,也有可能我理解不到位

#### org.elasticsearch.rest.RestController#dispatchRequest
```java
if (request.rawPath().equals("/favicon.ico")) {
    handleFavicon(request, channel);
    return;
}
try {
    tryAllHandlers(request, channel, threadContext);
} catch (Exception e) {
    
}

void tryAllHandlers(final RestRequest request, final RestChannel channel, final ThreadContext threadContext) throws Exception {
    //do something

    // Loop through all possible handlers, attempting to dispatch the request
    //
    Iterator<MethodHandlers> allHandlers = getAllHandlers(request);
    for (Iterator<MethodHandlers> it = allHandlers; it.hasNext(); ) {
        final Optional<RestHandler> mHandler = Optional.ofNullable(it.next()).flatMap(mh -> mh.getHandler(request.method()));
        requestHandled = dispatchRequest(request, channel, client, mHandler);<1>
        if (requestHandled) {
            break;
        }
    }

    // If request has not been handled, fallback to a bad request error.
    if (requestHandled == false) {
        handleBadRequest(request, channel);
    }
}
```
1. 最终会调用到org.elasticsearch.rest.RestHandler#handleRequest,即请求各个rest请求的业务类