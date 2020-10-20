---
layout: post
title: "延时任务之DelayQueue"
subtitle: "DelayQueue使用和原理"
date: 2020-9-20
author: liying
category: http
tags: [java,DelayQueue]
finished: true
---

## DelayQueue

DelayQueue一般用于延时任务，用一个无界队列来实现，

```
@PostConstruct
public void dropConsumer() {
    executor.submit(new Runnable() {
        @Override
        public void run() {
            while (true) {
                try {
                    ConsumerDelayDropTask task = delayDropTasks.take();
                    consumerMap.remove(task.getBootStrap());
                    task.getConsumer().close();
                    logger.info("consumer {} removed,expired time={}", task.toString(), task.getTime() - task.getStart());
                } catch (InterruptedException e) {
                    logger.error("kafka consumer remove error,msg={}", e);
                }
            }
        }
    });
}
```



```
@Data
class ConsumerDelayDropTask implements Delayed {
    private KafkaConsumer consumer;
    private long start = System.currentTimeMillis();
    private long time;
    private String bootStrap;

    public ConsumerDelayDropTask(String bootStrap, KafkaConsumer consumer, long time) {
        this.bootStrap = bootStrap;
        this.consumer = consumer;
        this.time = time;
    }

    @Override
    public long getDelay(@NotNull TimeUnit unit) {
        return unit.convert((start + time) - System.currentTimeMillis(), TimeUnit.MILLISECONDS);

    }

    @Override
    public int compareTo(@NotNull Delayed o) {
        ConsumerDelayDropTask o1 = (ConsumerDelayDropTask) o;
        return (int) (this.getDelay(TimeUnit.MILLISECONDS) - o.getDelay(TimeUnit.MILLISECONDS));
    }

    @Override
    public String toString() {
        return "ConsumerDelayDropTask{" +
                "consumer='" + consumer.metrics().get("client-id") + '\'' +
                ", time=" + time +
                '}';
    }
}
```

```
delayDropTasks.add(new ConsumerDelayDropTask(cluster.getBootstraps(), consumer, TimeUnit.MINUTES.toMillis(1)));
```

```
/**
 * 关闭consumer 队列
 */
DelayQueue<ConsumerDelayDropTask> delayDropTasks = new DelayQueue<>();
/**
```