---
layout: post
title: Python知识点总结
---

# meta

## \_\_init\_\_

# decorator

## @classmethod

+ @classmethod: 当该方法被调用时，我们把Class作为第一个参数传递，而不是这个Class的实例，这
意味着你可以使用这个类以及它的properties而不是特定的类实例

## @staticmethod

+ @staticmethod: 当该方法被调用时，我们并不传递类实例，这意味着你可以将函数放入Class中，
但是你并不能访问类实例（当方法跟Class有关，但是又不需要实例时，特别有用）.



# pytest

# lambda


# daemon thread

## 官方解释
> A thread can be flagged as a “daemon thread”. The significance of this flag is that the entire Python program exits when only daemon threads are left
## 理解
+ non-daemon thread，当你没有为thread设置daemon=True时，就需要管理线程的生命周期。执行以下代码就会发现，该程序一直会执行，除非显式的join或者cancel（cancel并不能保证thread结束，有坑）。

```python
import threading


def printit():
    while True:
        import time
        time.sleep(1)
        print "Hello"

threading.Thread(target=printit).start()
```

+ daemon thread, 当你设置daemon=True时，如果主线程退出时，daemon thread会随之退出

```python
import threading


def printit():
    while True:
        import time
        time.sleep(1)
        print "Hello"

threading.Thread(target=printit, daemon=True).start()
```
