---
layout: post
title: 开源进展梳理
---
# 背景/问题
现有的线程模型，不同请求间相互影响
+ sendfile block住networkthread，导致其他请求无法被执行
    + 表现 ResponseQueue偏高，ResponseTime偏高
+ request-queue打满block住networkthread，request请求block住了response
    + 表现 LocalTime/RequestQueue/ResponseQueue/ResponseTime均升高
# 目标
改进线程模型，避免不同请求间的相互影响
+ 针对问题1，fetch请求和其他请求隔离
+ 针对问题2，req和resp隔离，分别注册read 和write事件，要解决时序问题
# 方案
## 问题1
+ 方案1-sendfile 调整到 io.thread中执行，扩容io.thread减轻问题
+ 方案2-sendfile调整到独立线程执行
    + 每次读请求都新建一个线程代价太大，参考hdfs,rocketmq,netty以及nginx如何实现
        + hdfs
            + 多线程模型 
                + 单个acceptor监听connecting事件
                + 使用SelectorPoll监听Writable事件
            + SingleSelector+Threadpool模型 
                + 单个selector监听writable事件
                + 多线程完成数据发送，非阻塞时返回
        + rocketmq
            + 物理存储模型决定了rocketmq并不适合使用sendfile，因为group commit机制会导致消费单个分区也成了随机读，那么如果触发磁盘读，会有大量的随机磁盘读
            + 题外话，rocketmq如何避免磁盘读
                + 实时读取大概率在pagecache
                + lag读建议读follower，因为rocketmq的follower上不存在master，因此不存在污染其他master的问题
        + netty-FileRegion什么时候使用，待深入研究
        + nginx-
    + ThreadPoolExecutor
        + WorkQueue
    + 遗留问题
        + 实时读，seperate thread性能反而会下降，因为qps升高，每次遍历的请求变多，如何加速，如何分到16个线程是不是就没有问题
        + select的超时时间精确性无法保证-恢复300观察
            - 恢复300后，毛刺消除，原因？
        + iotime为什么有时候会有毛刺，问题在哪里
        + ProduceResponseQueueTime为什么会高于FetchConsumerResponseQueueTime，按说受影响是一样的
            + 难道是ProduceLocalTime影响？
            + 换正常的机器验证
    + todo
        + 完善调研
        + 解决遗留问题
        + 总结nio的使用姿势
# 实验步骤
写入流量 100MB/s
+ 1 备份leader
+ 2 部署1.3.9/par-metric
+ 3 恢复leader
+ 4 验证指标

+ 1.3.9 no consumer
    + 0922-0952
+ 



# 实验场景
+ follower_jd
    + 跨机房传输耗时不可忽略
+ 结论
    + 99线ResponseQueue符合预期下降
    + 999线也有下降，但是毛刺的原因是什么
    + LocalTime为什么升高 -> 读没有限速导致磁盘读严重 -> 影响写
+ 18-10:00
    + 16个网络线程 将下游作业数减少至10个，读盘性能显著题提升
        + ioutil显著下降，ioutil确实能反应磁盘性能
            + 看起来ioutil没有打满，不会影响读写？
        + msec显著下降
        + response_queue_time显著下降
    + 统计单盘并发
    + 猜测 
        + 如果读性能到瓶颈，写也会受影响，但是观测写磁盘性能并无大的波动
            + raid这一层有瓶颈？
            + 内存有瓶颈?如果内存有瓶颈？
            + 但是以上二者有瓶颈，受影响的难道不是所有盘么
            + 如果说hdd的io.util的确能反应磁盘瓶颈，线上出现慢节点的原因是什么？msec_read/msec_write是否能反应磁盘性能达到瓶颈
            + 还是要统计单盘并发以及延时
        + SocketServerMetricIORatio受读写请求影响
            + 写流量上涨，单次receive所花时间也会随之上涨
                + 一次只读一个StageReceive
            + 磁盘读，会block住 selector的poll流程
                + 在独立线程执行sendfile
    + todo 
        + 慢节点分析
            + 高峰期/非高峰期
            + 主要因素是LocalTime还是ResponseQueueTime
        + 指标细化
            + receive耗时统计
            + send耗时统计
            + 参考netty/brpc的指标建设
            