---
layout: post
title: ElasticSearch使用总结1-DSL
---
# Overview
ES提供了三种查询方式：SQL、DSL和不同语言的Client，这里总结下各种查询方式的用法

# DSL
两种DSL 请求体搜索和轻量搜索（ad-hoc）
## 请求体查询（Request Body）
query总体分为3部分，query, aggregations，下面将分开讲述如何使用
```json
{
  "query": {},
  "_source": {},
  "aggs": {
}
```
### 查询
+ 单个条件精确查询
```json
{
    "term" : {
        "price" : 20
    }
}
```
类似SQL的查询
```sql
SELECT document
FROM   products
WHERE  price = 20
```
+ 组合查询
如果希望通过多个值过滤，比如类似以下的SQL语句
```sql
SELECT product
FROM   products
WHERE  (price = 20 OR productID = "XHDK-A-1293-#fJ3")
  AND  (price != 30)
```
那么就需要使用bool过滤器
```json
{
   "query" : {
      "bool" : { 
         "filter" : {
            "bool" : {
              "should" : [
                 { "term" : {"price" : 20}}, 
                 { "term" : {"productID" : "XHDK-A-1293-#fJ3"}} 
              ],
              "must_not" : {
                 "term" : {"price" : 30} 
              }
           }
         }
      }
   }
}
```

#### 常用查询
+ match 模糊匹配
  + match_all 
  + match 
  + multi_match 多字段执行相同的查询
  + match_phrase 暂时不是太明确和term的区别
    + The match_phrase query analyzes the text and creates a phrase query out of the analyzed text.
    + A phrase query matches terms up to a configurable slop (which defaults to 0) in any order.
+ range
+ term 精确值查询
+ terms 和term类似，并且支持多值匹配
```json
{ "terms": { "tag": [ "search", "full_text", "nosql" ] }}
```
+ exists and missing
#### filter vs query
+ filter通常情况下比query性能更好
+ 查询在倒排索引的优化下，在匹配少量文档时，可能与一个涵盖百万文档的filter表现一样好，但是过滤总体表现更稳定
+ 使用查询进行全文搜索或者相关性得分搜索，除此以外，均使用过滤
+ 优化手段：将query移到filter中，就会转换成一个不评分查询，此时就可以使用各种filter的优化手段提高性能
#### 语法验证
+ /gb/tweet/_validate/query
+ /_validate/query?explain
#### 配置
+ adjust_pure_negative
+ boost 和权重有关
### 聚合
聚合的基本语法如下
```json
"aggregations" : {
    "<aggregation_name>" : {
        "<aggregation_type>" : {
            <aggregation_body>
        }
        [,"meta" : {  [<meta_data_body>] } ]?
        [,"aggregations" : { [<sub_aggregation>]+ } ]?
    }
    [,"<aggregation_name_2>" : { ... } ]*
}
```
#### 基本概念
需要理解两个基本概念：Buckets和Metrics
+ Buckets 类似SQL中的Group By
+ Metrics 类似COUNT,SUM,AVG等统计方法
```
SELECT COUNT(color) 
FROM table
GROUP BY color
``` 
COUNT(color)相当于Metrics，color相当于Buckets
#### 常用聚合
+ Term
+ 直方图
+ 柱状图
#### 聚合和过滤
+ 搜索过滤 先过滤后聚合，同时影响搜索结果和聚合结果
+ 聚合过滤 聚合时使用过滤条件，只影响聚合
+ 后过滤（post-filter） 只过滤搜索结果，不过滤聚合结果
#### 多桶排序
使用时再了解
#### 近似聚合
+ percentiles
+ cardinality
Elasticsearch 提供的首个近似聚合是 cardinality （注：基数）度量。 它提供一个字段的基数，即该字段的 distinct 或者 unique 值的数目
```sql
SELECT COUNT(DISTINCT color)
FROM cars
```
#### 优化聚合查询
结合现实场景选用合理的搜索算法
```json
{
  "aggs" : {
    "actors" : {
      "terms" : {
         "field" :        "actors",
         "size" :         10,
         // 改变搜索算法
         "collect_mode" : "breadth_first" 
      },
      "aggs" : {
        "costars" : {
          "terms" : {
            "field" : "actors",
            "size" :  5
          }
        }
      }
    }
  }
}
```
+ 默认深度优先 
  + 适用于总分组数远小于总数
  + 比如有100条数据，分组数时2
  + 每个分组50条数据
+ 广度优先 
  + 适用于每个组的数量远小于当前组总数
  + 比如有100条数据，分组数时50
  + 每个分组2条数据

### 分页
当返回结果的数量超过es设置的窗口数量（比如5w）时，就需要使用分页查询。分页查询的使用方式：先获取scroll_id，然后使用scroll_id查询

1 首先获取scroll_id
```

GET /old_index/_search?scroll=1m 
{
    "query": { "match_all": {}},
    "sort" : ["_doc"], 
    "size":  1000
}
```
2 基于scroll_id查询
```
GET /_search/scroll
{
    "scroll": "1m", 
    "scroll_id" : "cXVlcnlUaGVuRmV0Y2g7NTsxMDk5NDpkUmpiR2FjOFNhNnlCM1ZDMWpWYnRROzEwOTk1OmRSamJHYWM4U2E2eUIzVkMxalZidFE7MTA5OTM6ZFJqYkdhYzhTYTZ5QjNWQzFqVmJ0UTsxMTE5MDpBVUtwN2lxc1FLZV8yRGVjWlI2QUVBOzEwOTk2OmRSamJHYWM4U2E2eUIzVkMxalZidFE7MDs="
}
```


## 轻量搜索（Ad-hoc）
es还提供了一种简易的查询方式，不用写复杂的嵌套查询，使用时再总结

# Client
# java
## 分页
Spring提供了一个支持分页的api-stream，内部实现实际上是http请求scroll的封装
```java
  public <T> CloseableIterator<T> stream(SearchQuery query, Class<T> clazz) {
    return stream(query, clazz, resultsMapper);
  }

  public void use() {
    CloseableIterator<EsNoonPeakJobMetric> iterator = elasticsearchTemplate.stream(searchQuery, EsNoonPeakJobMetric.class);
    Map<Long, EsNoonPeakJobMetric> topoMetricMap = new HashMap<>();
    while (iterator.hasNext()) {
        EsNoonPeakJobMetric one = iterator.next();
        topoMetricMap.put(one.getTopologyId(), one);
    }
  }

```

# SQL
和通用的SQL语法类似，只不过不支持部分语法，不支持列表如下：
+ distinct


## 参考
+ https://www.elastic.co/guide/cn/elasticsearch/guide/current/query-dsl-intro.html