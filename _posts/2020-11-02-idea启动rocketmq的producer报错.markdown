```
layout: post
title: "idea启动rocketmq的producer报错"
subtitle: "java.lang.IllegalStateException: org.apache.rocketmq.remoting.exception.RemotingConnectException: connect to null failed"
date: 2020-11-02
author: liying
category: rocketmq
tags: rocketmq
finished: true
```

[toc]

## 解决方法

网上搜索RemotingConnectException出现的都是连接到某个ip:port失败，而出现null则说明是配置问题。

是由updateTopicRouteInfoFromNameServer方法抛出的错误。检查下nameserver配置。

```java
java.lang.IllegalStateException: org.apache.rocketmq.remoting.exception.RemotingConnectException: connect to null failed
	at org.apache.rocketmq.client.impl.factory.MQClientInstance.updateTopicRouteInfoFromNameServer(MQClientInstance.java:679)
	at org.apache.rocketmq.client.impl.factory.MQClientInstance.updateTopicRouteInfoFromNameServer(MQClientInstance.java:509)
	at org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl.tryToFindTopicPublishInfo(DefaultMQProducerImpl.java:693)
	at org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl.sendDefaultImpl(DefaultMQProducerImpl.java:557)
	at org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl.send(DefaultMQProducerImpl.java:1343)
	at org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl.send(DefaultMQProducerImpl.java:1289)
	at org.apache.rocketmq.client.producer.DefaultMQProducer.send(DefaultMQProducer.java:325)
	at org.apache.rocketmq.example.quickstart.Producer.main(Producer.java:67)
```

参考：[RocketMQ入门到入土（一）](https://www.lagou.com/lgeduarticle/138953.html )解决问题

提到使用tool.sh启动producer时，通过`export NAMESRV_ADDR=localhost:9876`解决问题。

所以尝试在idea的Producer类启动配置中，增加![image-20201102115105358](../img/mq-producer-conf.png)

应用后，再启动。

成功发送消息!