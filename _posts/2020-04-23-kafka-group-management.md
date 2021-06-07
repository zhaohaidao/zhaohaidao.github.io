---
layout: post
title: kafka消费组管理
---
# 前言
旧版有什么问题
用户有什么需求
设计目标是什么
# consumer group life circle
具体实现和proposal的区别
- 引入了empty-帮助用户更好的理解当前group状态
- 所有状态都可以转移到 dead-why

# 参考
https://cwiki.apache.org/confluence/display/KAFKA/Kafka+Client-side+Assignment+Proposal
https://github.com/apache/kafka/pull/165
https://github.com/apache/kafka/pull/1427
http://matt33.com/2017/10/22/consumer-join-group/
