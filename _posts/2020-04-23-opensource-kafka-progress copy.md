---
layout: post
title: 开源进展梳理
---
# 背景
记录参与的开源工作, 方便追踪各部分的进展
# kafka
# DOING
+ KIP-588 Allow producers to recover gracefully from transaction timeouts
    + KAFAKA-9910: https://github.com/apache/kafka/pull/9311
    + KAFKA-10504: https://github.com/apache/kafka/pull/9310
+ KIP-584 Versioning scheme for features
    + DOING: background study
        + client discovery
            + 对于仅依赖于使用特定版本的客户端逻辑，apiversion是足够的
            + 对于要求客户端行为有所不同的逻辑，apiversion是不够的
                + 比如：stream client需要开启新功能-升级到新的线程模型版本，但是这需要建立在broker所有节点均支持新线程模型的前提下，现在由人确定，容易出错
        + feature gating
            + 现阶段使用IBP控制broker的rpc行为，但是有时候需要做两次滚动升级
                + 第一次升级新版本
                + 第二次更新IBP（为什么不能一次性更新：自己的理解，降低upgrade风险，每次只改变一个变量）
                    + 第一次，只升级新版本，保证新代码对原有数据兼容
                    + 第二次，更新IBP，开启新feature
    + TODO: figure out how apiversion{Request/Response} works
        - DONE figure out how apiversion{Request/Response} works
        - TODO figure out how group rebalance works
            - 输出文档
                + DONE 状态机
                + 源码分析
            - 任务拆解 
                - 初版
                    - 更新consumer group meta-引入featureflag
                    - leader
                        - 定期describe feature flag
                        - syncgroup 持久化
                    - follower
                        - syncgroup 接收feature flag
                        - re-join时上报自己使用的featureflag
                    - enable kip-447，收到新的feature flag，切换新的线程模型
                - 疑问
                    - 弄清楚KIP-447如何工作，切入点在哪
                        - 新的线程模型，一个stream thread只会起1个producer，无法通过initTxn做fence，消费pendingoffset会造成重复，kip-447碰到pendingoffset会等待旧的txn超时abort
                            - 疑问：rebalance时，producer为啥不能initTxn
                        - 切入点：和process_gurantee_configs类似
                    - 弄清楚group rebalance中的consumer meta长什么样，怎么加feature flag
                    - 弄清楚consumer member怎么解析group coordinator
                        - 如果元数据广播是最终一致，那么什么时候启用新的线程模型
                        - 当member收到leader的feature flag
            - DONE 弄清楚leader如何trigger rebalance
                - KAFKA-6145: KIP-441 Pt. 6 Trigger probing rebalances until group is stable (#8409)
                    - 通过引用传递变量
                    - todo：整理 group rebalance这块的代码
            - DONE 这里为什么会提到the second rebalance，跟downgrade有什么关系
                + 意思指的是上一次的feature flag还未同步到所有member之前，不会发起新一轮的feature flag
                + 那leader怎么知道上一轮feature flag已经完成呢
                    - 需要再深入看看group rebalance protocol，从现在了解的细节来看，leader并不知道
                    - 新一轮的rebalance时，member会通过joingroup上传memberinfo，此时leader就能感知到member的feature flag
            - DONE the leader of the groupwill not kick off another feature request
                + 这里的feature request到底指的是什么，在group meta里新增flag么，这tmd跟downgrade有什么关系
                    - 确实跟downgrade无关
                + leader什么时候知道members全部都更新成新的线程模型
                    - 还是要细看下group rebalance protocol
                + members什么时候更新成新的线程模型,如果member更新了线程模型，broker还是v1-v2怎么办
                    - 当member收到leader的feature flag，此时所有broker都应该支持v2，因此更新成新的线程模型并不是问题，只是cluster-wide finalized features是最终一致模型，此时可能读到stale info
    + KAFKA-9689
# TODO
+ 数据读写
    + 内核优化
        + non-blocking network-io thread
            + 网络线程通过mute来主动反压，高版本已经支持，可以patch到低版本
            + pre-warm
            + seperate-thread https://issues.apache.org/jira/browse/HDFS-223?attachmentSortBy=dateTime
        + request-handler多队列
        + 全异步刷盘
        + 限流 请求/topic/分区/broker/磁盘
        + 在kafka应用层引入缓存，隔离实时读和回溯读
        + replica sync and reassignment in seperate thread
        + 分层存储时，新的follower可以不用做catch-up read
    + fetcher thread 的cpu使用率过高
+ bugfix
    + DONE 客户端 无效brokers.size>0注释删除
    + sensor.percentiles内存泄漏问题
    + io exception引起cleaner线程挂掉
+ stream
    + DOING reopen 9910
    + 系统性整理stream的设计和实现，输出源码分析
    + 整理group rebalance
        + 0.9
        + static membership
        + incremental rebalance
    + KIP-609 Use Pre-registration and Blocking Calls for Better Transaction Efficiency
    + KAFKA-9878: https://issues.apache.org/jira/browse/KAFKA-9878
+ 集群管理
    + 系统性整理controller的设计和实现，输出源码分析
        + zookeeper版本
        + raft版本
    + 元数据广播协议优化
        + gossip协议
        + 增量广播
        + rate limit for sendThread
    + 学习chubby以及其他的控制节点，看看能做什么其他优化
    + 启动优化-backup
        + 作为zk的observer，订阅事件
        + watch所有节点并调用更新
    + 元数据管理
        + 分区缩容
    + raft
# pulsar
# dragonboat
+ udp support
+ pre-vote support
# storm
# flink