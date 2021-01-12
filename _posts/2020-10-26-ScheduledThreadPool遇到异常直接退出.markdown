---
layout: post
title: "ScheduledThreadPool执行周期性任务时异常退出"
subtitle: "使用ScheduledThreadPool要在最外层catch异常，否则任务会悄悄消失"
date: 2020-10-26
author: liying
category: java
tags: [java,线程池]
finished: true
---

* 目录
{:toc #markdown-toc}
## 问题

定时处理资源告警的scheduleAtFixedRate定时任务丢失，没有继续执行。

## 定位

1.查看堆栈：

![java-stack](../img/java-stack.png)

正常情况下，ScheduledThreadPoolExecutor在任务执行过程中为TIMED_WAITING状态，而WAITING状态说明当前线程已经没有在执行任务了。通过堆栈信息发现是ScheduledThreadPoolExecutor的DelayedWorkQueue的take方法使得线程进入休眠。下面是RunnableScheduledFuture的源码。

```java
public RunnableScheduledFuture<?> take() throws InterruptedException {
            final ReentrantLock lock = this.lock;
            lock.lockInterruptibly();
            try {
                for (;;) {
                    RunnableScheduledFuture<?> first = queue[0];
                    if (first == null)
                        available.await();
                    else {
                        long delay = first.getDelay(NANOSECONDS);
                        if (delay <= 0)
                            return finishPoll(first);
                        first = null; // don't retain ref while waiting
                        if (leader != null)
                            available.await();
                        else {
                            Thread thisThread = Thread.currentThread();
                            leader = thisThread;
                            try {
                                available.awaitNanos(delay);
                            } finally {
                                if (leader == thisThread)
                                    leader = null;
                            }
                        }
                    }
                }
            } finally {
                if (leader == null && queue[0] != null)
                    available.signal();
                lock.unlock();
            }
        }
```

可以看出

```java
if (first == null)
    available.await();
```

由于获取不到任务，导致了线程休眠，所以问题是没有向队列中写入任务，导致消费者获取不到任务。

而真正实现任务的是ScheduledFutureTask中的run方法。

```java
/**
  * Overrides FutureTask version so as to reset/requeue if periodic.
 */
public void run() {
    boolean periodic = isPeriodic();
    if (!canRunInCurrentRunState(periodic))
        cancel(false);
    else if (!periodic)
        ScheduledFutureTask.super.run();
    else if (ScheduledFutureTask.super.runAndReset()) {
        setNextRunTime();
        reExecutePeriodic(outerTask);
    }
}
```

前面两个if对执行条件进行了判断，ScheduledFutureTask.super.runAndReset()去执行任务并返回结果。如果执行成功，才会去设置下一次的运行时间，执行下一次任务。

```java
    protected boolean runAndReset() {
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return false;
        boolean ran = false;
        int s = state;
        try {
            Callable<V> c = callable;
            if (c != null && s == NEW) {
                try {
                    c.call(); // don't set result
                    ran = true;
                } catch (Throwable ex) {
                    setException(ex);
                }
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
        return ran && s == NEW;
    }
```

runAndReset()方法使用callable来执行任务，如果call方法抛出了异常，那么ran=false，则runAndReset()返回的值也是false。所以之后的方法都不会被放进队列里，自然就导致了线程拿不到任务从而休眠。



## 总结

使用定时调度线程池的scheduleAtFixedRate和scheduleAtFixedDelay时，一定要在执行内容的最外层catch异常，否则当执行线程捕获到异常后，会将任务暂停，停止写入任务，导致整个定时任务失效。