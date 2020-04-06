---
layout: post
title: chrome developer tools中的source tab hang住问题
---
# 问题
当给某个超大的js文件添加断点后，后续想挑事，切换到source tab时，发现整个source tab会hang住，点按均不起作用
# 原因
给超大的js文件添加断点，会导致chrome会解析整个js文件，最终hang住。按理说，只要消除断点后，就不会再hang住。但是由于整个source tab都hang住，取消断点的按钮也无反应，下次切换到source tab仍然会继续hang住。
# 解决
1. 打开dev tools
2. 打开设置-restore defaults and reload
3. 再切换到source tab，此时发现之前的断点均已消失。
