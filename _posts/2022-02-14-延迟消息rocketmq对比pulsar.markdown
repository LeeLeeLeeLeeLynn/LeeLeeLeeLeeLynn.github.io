---
layout: post
title: "RocketMQ和Pulsar的延迟消息机制对比"
subtitle: "RocketMQ和Pulsar延迟消息支持的类型不同，具体内部实现区别是什么呢？"
date: 2022-02-14
author: liying
category: mq
tags: [mq,RocketMQ,Pulsar]
finished: true
---





* 目录
{:toc #markdown-toc}


### 延迟消息介绍

概念：延迟消息是指生产者发送消息发送消息后，不能立刻被消费者消费，需要等待指定的时间后才可以被消费。

场景：30分钟内支付完成，快到期时发送提醒；用户超过15天未登录时，给该用户发送召回推送。

实现思路：在消息中间件中，设置临时存储和延迟服务模块，到期后再投递到真实的用户主题中去。



![delay-msg](/Users/liying/Documents/个人博客/LeeLeeLeeLeeLynn.github.io/img/delay-msg.png)



### RocketMQ固定等级延迟消息

​	RocketMQ开源版本不支持任意精度的延迟消息，只支持固定刻度的延迟，默认支持18个等级的消息刻度。通过broker的参数配置。

```shell
messageDelayLevel=1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h
```

​	大致流程如下：

​	![RocketMQ延迟消息流转](../img/RocketMQ延迟消息流转.png)

**Step1:** RocketMQ 接收消息后，根据DelayTimeLevel判断是否是延迟消息，并将消息发送到延迟主题，根据level发送到不同的queue。

```java
// getDelayTimeLevel()>0 判断为延迟消息
if (msg.getDelayTimeLevel() > 0) {
  	//如果消息的延迟等级大于最大延迟等级，则设置为最大延迟等级
    if (msg.getDelayTimeLevel() > this.defaultMessageStore.getScheduleMessageService().getMaxDelayLevel()) {                    				msg.setDelayTimeLevel(this.defaultMessageStore.getScheduleMessageService().getMaxDelayLevel());
    }
		//消息发到延迟topic，名为SCHEDULE_TOPIC_XXXX
    topic = TopicValidator.RMQ_SYS_SCHEDULE_TOPIC;
  	// 根据不同level,发送到不同的queue，queueId为delayLevel-1
    queueId = ScheduleMessageService.delayLevel2QueueId(msg.getDelayTimeLevel());

    // 将真实的topic和queue存到消息的properties中，保证在流转中不丢失
    MessageAccessor.putProperty(msg, MessageConst.PROPERTY_REAL_TOPIC, msg.getTopic());
    MessageAccessor.putProperty(msg, MessageConst.PROPERTY_REAL_QUEUE_ID, String.valueOf(msg.getQueueId()));
    msg.setPropertiesString(MessageDecoder.messageProperties2String(msg.getProperties()));
		
  //设置成延迟topic和生成的queueId
    msg.setTopic(topic);
    msg.setQueueId(queueId);
}
```

SCHEDULE_TOPIC_XXXX这个topic是一个特殊的topic，和正常的topic不同的地方是：

1. 不会创建TopicConfig，因为也不需要consumer直接消费这个topic下的消息
2. 不会将topic注册到namesrv
3. 这个topic的队列个数和延时等级的个数是相同的



**step2:** 处理延时消息的服务是ScheduleMessageService，broker启动的时候DefaultMessageStore会调用org.apache.rocketmq.store.schedule.ScheduleMessageService#start来启动处理延时消息队列的服务：源码如下：

```java
	public void start() {
        if (started.compareAndSet(false, true)) {
          	//创建定时器
            this.timer = new Timer("ScheduleMessageTimerThread", true);
          	//遍历每一个延迟级别，创建对应的DeliverDelayedMessageTimerTask
            for (Map.Entry<Integer, Long> entry : this.delayLevelTable.entrySet()) {
                Integer level = entry.getKey();
                Long timeDelay = entry.getValue();
                Long offset = this.offsetTable.get(level);
                if (null == offset) {
                    offset = 0L;
                }

                if (timeDelay != null) {
                    this.timer.schedule(new DeliverDelayedMessageTimerTask(level, offset), FIRST_DELAY_TIME);
                }
            }

            this.timer.scheduleAtFixedRate(new TimerTask() {

                @Override
                public void run() {
                    try {
                      	//持久化offsetTable（刷盘，写入delayOffset.json）
                        if (started.get()) ScheduleMessageService.this.persist();
                    } catch (Throwable e) {
                        log.error("scheduleAtFixedRate flush exception", e);
                    }
                }
            }, 10000, this.defaultMessageStore.getMessageStoreConfig().getFlushDelayOffsetInterval());
        }
    }
```

DeliverDelayedMessageTimerTask是一个TimerTask，启动以后不断处理延时队列中的消息，直到出现异常则终止该线程重新启动一个新的TimerTask

```java
class DeliverDelayedMessageTimerTask extends TimerTask {
        private final int delayLevel;
        private final long offset;

        public DeliverDelayedMessageTimerTask(int delayLevel, long offset) {
            this.delayLevel = delayLevel;
            this.offset = offset;
        }

        @Override
        public void run() {
            try {
                if (isStarted()) {
                    this.executeOnTimeup();
                }
            } catch (Exception e) {
                // XXX: warn and notify me
                log.error("ScheduleMessageService, executeOnTimeup exception", e);
                ScheduleMessageService.this.timer.schedule(new DeliverDelayedMessageTimerTask(
                    this.delayLevel, this.offset), DELAY_FOR_A_PERIOD);
            }
        }

```

**step2.1:** executeOnTimeup方法

```java

	public void executeOnTimeup() {
   					//根据delayLevel选择对应的队列
            ConsumeQueue cq =              ScheduleMessageService.this.defaultMessageStore.findConsumeQueue(TopicValidator.RMQ_SYS_SCHEDULE_TOPIC,
                    delayLevel2QueueId(delayLevel));
						//异常恢复时的起始offest
            long failScheduleOffset = offset;

            if (cq != null) {
              	//从offset开始读取MappedFile
                SelectMappedBufferResult bufferCQ = cq.getIndexBuffer(this.offset);
                if (bufferCQ != null) {
                    try {
                        long nextOffset = offset;
                        int i = 0;
                        ConsumeQueueExt.CqExtUnit cqExtUnit = new ConsumeQueueExt.CqExtUnit();
                        for (; i < bufferCQ.getSize(); i += ConsumeQueue.CQ_STORE_UNIT_SIZE) {
                          	//consumeQueue中存储的一条消息对应的信息，格式见下图
                          	//物理偏移量
                            long offsetPy = bufferCQ.getByteBuffer().getLong();
                          	//消息大小
                            int sizePy = bufferCQ.getByteBuffer().getInt();
                          	//tagsCode，延迟消息的tagsCode是时间戳
                            long tagsCode = bufferCQ.getByteBuffer().getLong();
														//Check {@code tagsCode} is address of extend file or tags code.
                            if (cq.isExtAddr(tagsCode)) {
                              	//获取
                                if (cq.getExt(tagsCode, cqExtUnit)) {
                                    tagsCode = cqExtUnit.getTagsCode();
                                } else {
                                  //获取不到，去commitLog查存储时间加上延迟时间
                                    //can't find ext content.So re compute tags code.
                                    log.error("[BUG] can't find consume queue extend file content!addr={}, offsetPy={}, sizePy={}",
                                        tagsCode, offsetPy, sizePy);
                                    long msgStoreTime = defaultMessageStore.getCommitLog().pickupStoreTimestamp(offsetPy, sizePy);
                                    tagsCode = computeDeliverTimestamp(delayLevel, msgStoreTime);
                                }
                            }

                            long now = System.currentTimeMillis();
                          	//校验投递时间戳
                            long deliverTimestamp = this.correctDeliverTimestamp(now, tagsCode);
														//下一个要投递的消息的offset
                            nextOffset = offset + (i / ConsumeQueue.CQ_STORE_UNIT_SIZE);
														//计算距离投递还有多久
                            long countdown = deliverTimestamp - now;
														//已经到时间
                            if (countdown <= 0) {
                                MessageExt msgExt =
                                    ScheduleMessageService.this.defaultMessageStore.lookMessageByOffset(
                                        offsetPy, sizePy);

                                if (msgExt != null) {
                                    try {
                                        MessageExtBrokerInner msgInner = this.messageTimeup(msgExt);
                                      //事务half消息不支持延迟，直接丢弃消息
                                        if (TopicValidator.RMQ_SYS_TRANS_HALF_TOPIC.equals(msgInner.getTopic())) {
                                            log.error("[BUG] the real topic of schedule msg is {}, discard the msg. msg={}",
                                                    msgInner.getTopic(), msgInner);
                                            continue;
                                        }
                                      //
                                        PutMessageResult putMessageResult =
                                            ScheduleMessageService.this.writeMessageStore
                                                .putMessage(msgInner);
																				//投递成功，判断下一条
                                        if (putMessageResult != null
                                            && putMessageResult.getPutMessageStatus() == PutMessageStatus.PUT_OK) {
                                            continue;
                                        } else {
                                            // XXX: warn and notify me
                                            log.error(
                                                "ScheduleMessageService, a message time up, but reput it failed, topic: {} msgId {}",
                                                msgExt.getTopic(), msgExt.getMsgId());
                                          //DELAY_FOR_A_PERIOD后重试
                                            ScheduleMessageService.this.timer.schedule(
                                                new DeliverDelayedMessageTimerTask(this.delayLevel,
                                                    nextOffset), DELAY_FOR_A_PERIOD);
                                          //更新offset
                                            ScheduleMessageService.this.updateOffset(this.delayLevel,
                                                nextOffset);
                                          //结束方法
                                            return;
                                        }
                                    } catch (Exception e) {
                                        /*
                                         * XXX: warn and notify me



                                         */
                                        log.error(
                                            "ScheduleMessageService, messageTimeup execute error, drop it. msgExt="
                                                + msgExt + ", nextOffset=" + nextOffset + ",offsetPy="
                                                + offsetPy + ",sizePy=" + sizePy, e);
                                    }
                                }
                            } else {
                              	//计算多久后下一条消息到期，countdown时间后再次触发任务
                                ScheduleMessageService.this.timer.schedule(
                                    new DeliverDelayedMessageTimerTask(this.delayLevel, nextOffset),
                                    countdown);
                              	//更新nextOffset
                                ScheduleMessageService.this.updateOffset(this.delayLevel, nextOffset);
                              	//结束方法
                                return;
                            }
                        } // end of for
												//记录并更新nextOffset=起始offset+条目大小*条目数
                        nextOffset = offset + (i / ConsumeQueue.CQ_STORE_UNIT_SIZE);
                      	//本批次全部处理完，DELAY_FOR_A_WHILE后再重试，看是否有新消息写入
                        ScheduleMessageService.this.timer.schedule(new DeliverDelayedMessageTimerTask(
                            this.delayLevel, nextOffset), DELAY_FOR_A_WHILE);
                        ScheduleMessageService.this.updateOffset(this.delayLevel, nextOffset);
                        return;
                    } finally {

                        bufferCQ.release();
                    }
                } // end of if (bufferCQ != null)
                else {
										//如果根据offsetTable中的offset没有找到对应的消息(可能被删除了)，则按照当前ConsumeQueue的最小offset开始处理
                    long cqMinOffset = cq.getMinOffsetInQueue();
                    if (offset < cqMinOffset) {
                        failScheduleOffset = cqMinOffset;
                        log.error("schedule CQ offset invalid. offset=" + offset + ", cqMinOffset="
                            + cqMinOffset + ", queueId=" + cq.getQueueId());
                    }
                }
            } // end of if (cq != null)
    
						//循环校验是否有消息到期，DELAY_FOR_A_WHILE = 100ms
            ScheduleMessageService.this.timer.schedule(new DeliverDelayedMessageTimerTask(this.delayLevel,
                failScheduleOffset), DELAY_FOR_A_WHILE);
        }
```

ConsumeQueue存储格式：

![img](https://github.com/apache/rocketmq/raw/master/docs/cn/image/rocketmq_design_7.png)

### Pulsar任意时间延迟消息

![img](https://pic2.zhimg.com/80/v2-b31f3312f96f8f8ca36feb4249ff34a5_1440w.jpg)



Pulsar 实现延迟消息投递的方式比较简单，所有延迟投递的消息会被 Delayed Message Tracker 记录对应的 index。index 是由 timestamp | LedgerID | EntryID 三部分组成，其中 LedgerID | EntryID 用于定位该消息，timestamp 除了记录需要投递的时间，还用于 delayed index 优先级队列排序。

Delayed Message Tracker 在堆外内存维护着一个 delayed index 优先级队列，根据延迟时间进行堆排序，延迟时间最短的会放在头上，时间越长越靠后。consumer 在消费时，会先去 Delayed Message Tracker 检查，是否有到期需要投递的消息，如果有到期的消息，则从 Tracker 中拿出对应的 index，找到对应的消息进行消费；如果没有到期的消息，则直接消费正常的消息。

![img](https://pic1.zhimg.com/80/v2-7ae9f2e63a47473170a1918d11fa0a04_1440w.jpg)

```java
public Set<PositionImpl> getScheduledMessages(int maxMessages) {
  	    //最大数量限制  
  			int n = maxMessages;
        Set<PositionImpl> positions = new TreeSet<>();
        long now = clock.millis();
        // Pick all the messages that will be ready within the tick time period.
        // This is to avoid keeping rescheduling the timer for each message at
        // very short delay
        long cutoffTime = now + tickTimeMillis;
				//	优先队列
        while (n > 0 && !priorityQueue.isEmpty()) {
            long timestamp = priorityQueue.peekN1();
            if (timestamp > cutoffTime) {
                break;
            }

            long ledgerId = priorityQueue.peekN2();
            long entryId = priorityQueue.peekN3();
            positions.add(new PositionImpl(ledgerId, entryId));

            priorityQueue.pop();
            --n;
        }

        if (log.isDebugEnabled()) {
            log.debug("[{}] Get scheduled messages - found {}", dispatcher.getName(), positions.size());
        }
        updateTimer();
        return positions;
    }
```

从 Pulsar 的延迟消息投递实现原理可以看出，该方法简单高效，对 Pulsar 内核侵入性较小，可以支持到任意时间的延迟消息。但同时发现，Pulsar 的实现方案无法支持大规模使用延迟消息，主要有以下两个原因：

**1.delayed index 队列受到内存限制**

一条延迟消息的 delayed index 由三个 long 组成，对于小规模的延迟消息来说，内存开销并不大。但由于 index 队列是 subscription 级别，对于 topic 的同一个 partition 来说，有多少个 subscription 就需要维护多少个 index 队列；同时，由于延迟消息越多、延迟的时间越长，index 队列内存占用也会更多。

**2.delayed index 队列重建时间开销**

上面有提到，如果集群出现 Broker 宕机或者 topic 的 ownership 转移，Pulsar 会重建 delayed index 队列。对于跨度时间长的大规模延迟消息，重建时间可能会到小时级别。为了减小 delayed index 队列重建时间，虽然可以给 topic 分更多的 partition 提高重建的并发度，但没有彻底解决重建时间开销问题。



**既然Pulsar支持任意时刻消息，用其负责45天及其以上消息延迟合适吗？**

由于Pulsar基于优先队列实现任意时刻消息，不同的时间戳的日志混合在一起存储。由于时间长的延迟没有ack导致整个日志块无法被删除。所以基于Pulsar管理过长时间的延迟并不合适。

相反，RocketMQ的调度固定时间等级单线程去运行，系统开销很小，同时避免了排序；将其两者结合实现最优解。



### Kafka+redis延迟消息实现方案



参考：

1. https://www.cnblogs.com/heihaozi/p/14266311.html
2. https://cloud.tencent.com/developer/article/1581368
3. https://blog.csdn.net/qq_40144701/article/details/112499892
3. https://zhuanlan.zhihu.com/p/353609068
3. https://www.cnblogs.com/sunshine-2015/p/9017426.html





