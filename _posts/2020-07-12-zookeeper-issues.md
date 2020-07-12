---
layout: post
title: zookeeper使用问题记录
---
# Overview
记录使用zookeeper遇到的问题和解决方法。
# 问题: Error contacting service. It is probably not running
## 根本原因
使用的zookeeper3.5.x版本因为jdk兼容性问题启动会失败。不过导致zookeeper未正常启动的原因有多种，这里着重描述分析过程。
## 分析
调用zkServer start显示zookeeper已经启动，但是启动kafka报连接zookeeper超时，调用zkCli也会报Connection Loss错误。另外调用zkServer status会报：
```
ZooKeeper JMX enabled by default
Using config: /usr/local/etc/zookeeper/zoo.cfg
Client port found: 2181. Client address: localhost.
Error contacting service. It is probably not running.
```
很明显，zookeeper没有正常启动，原因是什么呢，我们首先需要找到启动日志：通过lsof可以找到zookeeper进程的所有fd信息，其中就包括启动日志。调用lsof命令并搜索zookeeper.log即可了解启动日志所在目录
```
lsof -p 96601|grep zookeeper.log             
java    96601 zhaohaiyuan   46w      REG                1,4   130321         12893267446 /usr/local/var/log/zookeeper/zookeeper.log
```
打开zookeeper.log即可发现zookeeper在启动阶段抛出了异常：java.lang.NoSuchMethodError
```
java.lang.NoSuchMethodError: java.nio.ByteBuffer.flip()Ljava/nio/ByteBuffer;
        at org.apache.zookeeper.server.NIOServerCnxn.doIO(NIOServerCnxn.java:331)
        at org.apache.zookeeper.server.NIOServerCnxnFactory$IOWorkRequest.doWork(NIOServerCnxnFactory.java:530)
        at org.apache.zookeeper.server.WorkerService$ScheduledWorkRequest.run(WorkerService.java:155)
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
        at java.lang.Thread.run(Thread.java:745)
```
Google可知，使用zookeeper3.5.+的版本，会有jdk的兼容性问题，具体有两个解决办法。
+ 升级到jdk9
+ 降级zookeeper，使用3.4.+版本
详情可见：https://stackoverflow.com/questions/60612999/kafka-with-zookeeper-3-5-7-crash-nosuchmethoderror-java-nio-bytebuffer-flip/60613000#60613000

