---
layout:     post
title:      "elasticsearch对日期查询的优化"
date:       2019-08-07 17:03:00
author:     "聼雨夜"
catalog: true
tags:
    - elasticsearch
    - 源码
    - 搜索引擎
---
### 概述
设想一种场景,有一个时间戳属性**timestamp**,该属性值索引的最大值和最小值为
**max_timestamp**和**min_timestamp**,而查询范围值为**to小于min_timestamp或from大于max_timestamp**
这种情况下,其实是不用进行查询的,因为肯定没有文档能匹配上,对这个查询直接范围无结果即可,这样就提高了查询效率
和减少资源消耗
### 调用流程
1. org.elasticsearch.action.search.TransportSearchAction#doExecute(org.elasticsearch.tasks.Task, org.elasticsearch.action.search.SearchRequest, org.elasticsearch.action.ActionListener<org.elasticsearch.action.search.SearchResponse>)
2. org.elasticsearch.index.query.Rewriteable#rewriteAndFetch(T, org.elasticsearch.index.query.QueryRewriteContext, org.elasticsearch.action.ActionListener<T>)
3. org.elasticsearch.index.query.Rewriteable#rewriteAndFetch(T, org.elasticsearch.index.query.QueryRewriteContext, org.elasticsearch.action.ActionListener<T>, int)
4. org.elasticsearch.index.query.AbstractQueryBuilder#rewrite
5. org.elasticsearch.index.query.RangeQueryBuilder#doRewrite
6. org.elasticsearch.index.query.RangeQueryBuilder#getRelation
7. org.elasticsearch.index.mapper.DateFieldMapper.DateFieldType#isFieldWithinQuery
### 源码分析
>为了优化搜索条件,elasticsearch提供了方法**org.elasticsearch.index.query.AbstractQueryBuilder#rewrite**
>来对搜索条件进行重写

#### org.elasticsearch.index.query.AbstractQueryBuilder#rewrite

```java
@Override
public final QueryBuilder rewrite(QueryRewriteContext queryShardContext) throws IOException {
    //重写的核心逻辑
    QueryBuilder rewritten = doRewrite(queryShardContext);
    if (rewritten == this) {
        return rewritten;
    }
    if (queryName() != null && rewritten.queryName() == null) { // we inherit the name
        rewritten.queryName(queryName());
    }
    if (boost() != DEFAULT_BOOST && rewritten.boost() == DEFAULT_BOOST) {
        rewritten.boost(boost());
    }
    return rewritten;
}
```

#### org.elasticsearch.index.query.RangeQueryBuilder#doRewrite

```java
@Override
protected QueryBuilder doRewrite(QueryRewriteContext queryRewriteContext) throws IOException {
    //do some thing
    //查询范围值和该属性值范围(最小值/最大值的关系)
    //相交,不相交,包含
    final MappedFieldType.Relation relation = getRelation(queryRewriteContext);//(1)
    switch (relation) {
        //不相交
    case DISJOINT://(2)
        return new MatchNoneQueryBuilder();
        //包含
    case WITHIN://(3)
        if (from != null || to != null || format != null || timeZone != null) {
            RangeQueryBuilder newRangeQuery = new RangeQueryBuilder(fieldName);
            //既然是包含,则所有文档都能匹配上,则from设置为空
            newRangeQuery.from(null);
            //既然是包含,则所有文档都能匹配上,则to设置为空
            newRangeQuery.to(null);
            newRangeQuery.format = null;
            newRangeQuery.timeZone = null;
            return newRangeQuery;
        } else {
            return this;
        }
        //相交
    case INTERSECTS:
        return this;
    default:
        throw new AssertionError();
    }
}
```

* (1) 获取搜索值和属性的关系,**org.elasticsearch.index.mapper.MappedFieldType.Relation**
* (2) 不相交,即搜索值范围不在实际值范围,则rewrite为MatchNoneQueryBuilder,即没有匹配上任何文档的query
* (3) 包含情况下,该索引所有文档都能匹配上,那么设置**from和to**都为null,省去匹配的消耗

#### org.elasticsearch.index.query.RangeQueryBuilder#getRelation
* > 获取搜索值和属性的关系,**org.elasticsearch.index.mapper.MappedFieldType.Relation**

```java
/**
 * An enum used to describe the relation between the range of terms in a
 * shard when compared with a query range
 */
public enum Relation {
    //包含,搜索值存在于实际值范围
    WITHIN,
    //相交,搜索值和实际值范围存在交集
    INTERSECTS,
    //不相交,即搜索值不在实际值范围内
    DISJOINT;
}
```
#### org.elasticsearch.index.query.RangeQueryBuilder#getRelation
```java
// Overridable for testing only
protected MappedFieldType.Relation getRelation(QueryRewriteContext queryRewriteContext) throws IOException {
    QueryShardContext shardContext = queryRewriteContext.convertToShardContext();
    // If the context is null we are not on the shard and cannot
    // rewrite so just pretend there is an intersection so that the rewrite is a noop
    if (shardContext == null || shardContext.getIndexReader() == null) {
        return MappedFieldType.Relation.INTERSECTS;
    }
    final MapperService mapperService = shardContext.getMapperService();
    final MappedFieldType fieldType = mapperService.fullName(fieldName);
    if (fieldType == null) {
        // no field means we have no values
        return MappedFieldType.Relation.DISJOINT;
    } else {
        DateMathParser dateMathParser = getForceDateParser();
        //获取搜索条件和属性值范围关系
        //(1)
        return fieldType.isFieldWithinQuery(shardContext.getIndexReader(), from, to, includeLower,
                includeUpper, timeZone, dateMathParser, queryRewriteContext);
    }
}
```
* (1) 此处是获取**Relation**的关键代码,即调用org.elasticsearch.index.mapper.MappedFieldType#isFieldWithinQuery

#### org.elasticsearch.index.mapper.MappedFieldType#isFieldWithinQuery
```java
/** Return whether all values of the given {@link IndexReader} are within the range,
 *  outside the range or cross the range. The default implementation returns
 *  {@link Relation#INTERSECTS}, which is always fine to return when there is
 *  no way to check whether values are actually within bounds. */
public Relation isFieldWithinQuery(
    IndexReader reader,
    Object from, Object to,
    boolean includeLower, boolean includeUpper,
    DateTimeZone timeZone, DateMathParser dateMathParser, QueryRewriteContext context) throws IOException {
    //默认关系
    return Relation.INTERSECTS;
}
```
* 抽象类**MappedFieldType#isFieldWithinQuery**默认实现是相交
  即,不做任何处理
* 重写方法**isFieldWithinQuery**,目前只有一个实现类**DateFieldMapper**,接下来我们看代码实现
#### org.elasticsearch.index.mapper.DateFieldMapper.DateFieldType#isFieldWithinQuery
```java
@Override
public Relation isFieldWithinQuery(IndexReader reader, Object from, Object to, boolean includeLower, boolean includeUpper,
                                   DateTimeZone timeZone, DateMathParser dateParser,
                                   QueryRewriteContext context) throws IOException {
    //do something
    //实际索引值最小值
    long minValue = LongPoint.decodeDimension(PointValues.getMinPackedValue(reader, name()), 0);
    //实际索引值最大值
    long maxValue = LongPoint.decodeDimension(PointValues.getMaxPackedValue(reader, name()), 0);

    if (minValue >= fromInclusive && maxValue <= toInclusive) {//(1)
        return Relation.WITHIN;
    } else if (maxValue < fromInclusive || minValue > toInclusive) {//(2)
        return Relation.DISJOINT;
    } else {
        return Relation.INTERSECTS;
    }
}
```
* (1) 搜索值在实际值范围内,也就是包含
* (2) 搜索值在实际值范围外,也就是不相交,这种情况下是匹配不上任何文档的,可rewrite为**MatchNoneQueryBuilder**
