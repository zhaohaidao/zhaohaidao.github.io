---
layout: post
title: kafka-深入理解isr
---
# 前言
随着kafka使用越来越广泛，isr的问题逐渐暴露出来，本文记录了isr的演进过程，并且希望通过与paxos/raft对比，尝试理解isr的正确性
# isr演进
## 0.10.2
+ implement: truncate to HWM
+ fix: leader epoch
## KIP-101

## KIP-279
## KIP-320
## TLA+
# isr vs paxos/raft


