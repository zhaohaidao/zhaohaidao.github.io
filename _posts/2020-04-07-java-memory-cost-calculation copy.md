---
layout: post
title: java内存占用计算
---
# 摘要
做系统设计或者性能分析时，我们往往需
要评估和对比不同
todo
# 背景知识
给超大的js文件添加断点，会导致chrome会解析整个js文件，最终hang住。按理说，只要消除断点后，就不会再hang住。但是由于整个source tab都hang住，取消断点的按钮也无反应，下次切换到source tab仍然会继续hang住。
# 解决
1. 打开dev tools
2. 打开设置-restore defaults and reload
3. 再切换到source tab，此时发现之前的断点均已消失。
