---
layout: post
title: Kafka源码分析计划
---
# Overview
为了更好的理解Kafka，计划开展Kafka源码分析计划

# 总体目标
短期来说，为了能够更快的分析和解决问题
长期来说，为了建立系统性的知识体系

# 总体计划（先易后难）
## Overview
## Dependency
+ java nio
+ tcp
## Kafka Client
+ 网络模型
  + NIO
+ Consumer
  + 线程模型
  + GroupManagement
    + assign
  + OffsetManagement
    + reset
    + commit
  + 完整的消费流程
  + Metrics/Monitor
+ Producer
  + 线程模型
  + 完整的写入流程
  + Metrics/Monitor
+ Usage
  + Storm
    + spout
    + bolt
  + Flink kafka connector 
    + kafka source
      + thread model
      + Poll/ListOffset/PartitionDiscovery
    + kafka sink 
## Kafka Write/Read
## Kafka Meta Management
## EOS
## Raft-based controller

