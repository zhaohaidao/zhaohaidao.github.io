---
layout: post
title: kafka 事务-客户端部分
---
# 摘要
+ definitions: ask and answer questions
+ tools: monitoring logging alerting tracing

# 状态机
| 源状态   | 目的状态  | 触发条件 | 
|  ----  | ----  | ---- | 
| UNINTIALIZED  | INITIALIZING  | begin initialize |
| INITIALIZING  | READY| initialize succeed|
| READY  | IN_TRANSACTION | begin transaction
| READY  | UNINTIALIZED | reset produce id
| IN_TRANSACTION  | COMMITTING_TRANSACTION | begin commit
| IN_TRANSACTION  | ABORTING_TRANSACTION | begin abort
| IN_TRANSACTION  | ABORTABLE_ERROR | none-fatal error
| COMMITTING_TRANSACTION  | ABORTABLE_ERROR | none-fatal error
| ABORTING_TRANSACTION  | READY | completed txn without epoch bump
| ABORTING_TRANSACTION  | INITIALIZING | completed txn with epoch bump
| ABORTABLE_ERROR  | ABORTABLE_ERROR | none-fatal error
| ABORTABLE_ERROR  | ABORTING_TRANSACTION | begin abort
| ALL | FATAL_ERROR | fatal error
![avatar](../img/txn-state-machine(client-side).png)
# 名词解释
## Kafka Stream DSL
## KStreams
## KTables
## JOIN
## CO-PARTITION
repartition
co-partition
enforcePartitionNumber
inputTopic

## 计算


# 参考
https://docs.microsoft.com/en-us/dotnet/architecture/cloud-native/observability-patterns
https://www.youtube.com/watch?v=U4E0QxzswQc&feature=youtu.be
https://www.aternity.com/blogs/why-you-need-both-monitoring-and-observability/