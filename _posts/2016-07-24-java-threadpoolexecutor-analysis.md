---
layout: post
title: ThreadPoolExecutor解析
---
# 背景

最近使用ThreadPoolExecutor在后台跑任务时，发现当任务由于未知异常中断且异常未被显式捕捉时，没有任何错误提示。

因此决定好好研究下java的线程库, 为了搞清楚问题的根本原因，选择Thread和ThreadPoolExecutor两种方式启动线程执行任务。

## ExceptionTask

如代码所示，task中直接抛NullPointerException

```java
public void run() {
    throw new NullPointerException("just test");
}
```

## Simple Runnable

如代码所示，直接在run()模块内 throw NullPointerException, console将打印出异常的堆栈信息

```java
PrimeRun p = new PrimeRun(143);
new Thread(p).start();
```

## ThreadPool

见代码示例，task由ThreadPoolExecutor执行时，run()中throw NullPointerException,该任务静默停止，不会通知main线程有异常发生

```java
PrimeRun p = new PrimeRun(143);
ThreadPoolTest threadPoolTest = new ThreadPoolTest();
for (int i = 0; i < 10; i ++) {
    threadPoolTest.scheduleTask(p);
}
```

# 真相

ThreadPoolExecutor中的异常处理逻辑如下

+ 当task运行出现RuntimeException、Error、Throwable或者其他异常时，均向上抛出
+ 在最外层的finally中会以task完成结束,并不会向上继续抛出

因此，task执行中出现未知异常中断，默认情况下，主线程是不会收到task异常通知的。

```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
        while (task != null || (task = getTask()) != null) {
            w.lock();
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly);
    }
}
```
# 解决方案

我的应用场景在task中显式catch异常即可，不用做相关的清理工作，更优雅的处理方式，见以下链接

[foo]: <https://www.securecoding.cert.org/confluence/display/java/TPS03-J.+Ensure+that+tasks+executing+in+a+thread+pool+do+not+fail+silently>

# 小结

java中的Executor将任务的提交和执行以生产者-消费者模式解耦。其实调度、提交以及执行这三者也应该解耦，每一项都可以任意替换

+ java的并发库中的ThreadPoolExecutor其实就是个经典的例子：executor、task已经被解耦，调度并未耦合在其中
+ 可以参考ScheduledThreadPoolExecutor的方式，实现CustomedScheduledThreadPoolExecutor
