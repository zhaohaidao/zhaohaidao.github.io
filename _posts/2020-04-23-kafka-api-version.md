---
layout: post
title: Kafka双向兼容之ApiVersion
---
# 前言
Kafka具备双向客户端兼容性策略，换言之，新版本的client可以和旧版本的brokers通信，旧版本的client也可以和新版本的brokers通信。那么kafka是如何实现这一策略的呢？随着时间的推移，Kafka的消息格式会不断发生变化，因此，client和broker需要就其通信的消息格式达成一致。这是通过ApiVersion特性实现的。
# ApiVersion
- 什么是ApiVersion

ApiVersion通过两元组表达(ApiKey, ApiVersion)，client带着ApiVersion向broker发送request，broker也就能根据ApiVersion返回保证兼容性的response
    + ApiKey
        - Produce
        - Fetch
        - ListOffset
    + Version 
        - 0
        - 1
        - 2
- 解决什么问题

0.10版本以前，kafka客户端无法知道broker具体支持哪个版本的api。这意味着kafka客户端无法执行所需的功能，也无法返回有意义的错误给应用程序，导致客户端和应用程序难以支持多个版本的kafka，这将对kafka的生态系统造成显著影响：提高维护成本；影响商业化进展
- 如何解决：双端是如何通信保证双向兼容的
+ 1  client -> multi brokers ApiVersionRequest
+ 2  multi brokers -> client ApiVersionResponse
+ 3  client use latest version
+ 4  deprecation by doc
+ 5  when disconnect, client send ApiverVersionReq again
# 展望未来
- 存在的问题
client无法动态的感知到所有broker都升级到某个版本后，开启feature-x
现在只能手动更新
+ 1 更新所有brokers
+ 2 升级client
- 如何解决
KIP-584: Versioning scheme for features
# 参考
+ https://kafka.apache.org/protocol#api_versions