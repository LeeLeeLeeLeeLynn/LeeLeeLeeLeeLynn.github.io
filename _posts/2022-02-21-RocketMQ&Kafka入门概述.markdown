```
layout: post
title: "RocketMQ默认的队列选择策略"
subtitle: "RocketMQ是如何均衡消息发送队列的呢？如果遇到broker异常，是否可以跳过当前broker呢？"
date: 2022-02-21
author: liying
category: mq
tags: [mq,RocketMQ]
finished: true
```



[TOC]

### 消息队列 

> 消息队列，一般我们会简称它为MQ(Message Queue)

#### 概念

​	我们先不管消息(Message)这个词，来看看队列(Queue)。

​	![img](../img/2-1FG91032244Y.png)

​	java实现的队列：

<img src="../img/v2-17e40843cb64e2e2b6e2faf08f1fe717_720w.jpg" alt="java中的Queue" style="zoom:50%;" />

那为什么还需要MQ这种中间件呢？ 

消息队列可以简单理解为：**把要传输的数据放在队列中**。

![img](../img/v2-1c69d2be58358e9743e39130b41993a3_720w.jpg)

- 把数据放到消息队列叫做**生产者**
- 从消息队列里边取数据叫做**消费者**



#### 场景

​	为什么要用消息队列？

* 接耦

  * 多应用间通过消息队列对同一消息进行处理，避免调用接口失败导致整个过程失败；

    * 发送解耦

    ![解偶](../img/解偶.png)

    ​	避免日志收集服务不可用时，服务A和服务B的整个过程失败。

    * 消费解偶

    ![image-20220221175417185](/Users/liying/Documents/个人博客/LeeLeeLeeLeeLynn.github.io/img/接耦2.png)

    ​	避免增加服务接口时，修改代码。

* 异步

  * 多应用对消息队列中同一消息进行处理，应用间并发处理消息，相比串行处理，减少处理时间；

  ![异步](../img/异步.png)

* 削峰/限流

  * 广泛应用于秒杀或抢购活动中，避免流量过大导致应用系统挂掉的情况；

  ![image-20220221175119223](../img/削峰.png)

> "*当不需要立即获得结果，但是并发量又需要进行控制的时候，差不多就是需要使用消息队列的时候。*"



#### HA

* **高可用**

  无论是我们使用消息队列来做解耦、异步还是削峰，消息队列**肯定不能是单机**的。试着想一下，如果是单机的消息队列，万一这台机器挂了，那我们整个系统几乎就是不可用了。

  ![img](../img/v2-56f04a3376f0af3565e363890de22650_720w.jpg)

  要求消息队列是集群/分布式的

* **数据丢失问题**

  我们将数据写到消息队列上，系统B和C还没来得及取消息队列的数据，就挂掉了。**如果没有做任何的措施，我们的数据就丢了**。

![img](../img/v2-418113f16d65370c06e4c657b3497e59_720w.jpg)



#### 生产者/消费者模型

生产者&消费者：一对多的关系

![image-20220221192253218](../img/image-20220221192253218.png)

主题：逻辑概念，消息以**主题（Topic）**来分类，每一个主题都对应一个「消息队列」

### Kafka

#### 概念

* 分区

​	我们使用一个生活中的例子来说明：现在 A 城市生产的某商品需要运输到 B 城市，走的是公路，那么单通道的高速公路不论是在「A 城市商品增多」还是「现在 C 城市也要往 B 城市运输东西」这样的情况下都会出现「吞吐量不足」的问题。所以我们现在引入**分区（Partition）**的概念，类似“允许多修几条道”的方式对我们的主题完成了水平扩展。

![img](../img/v2-ba46359183324eb5d10f70d00736a4ce_720w.jpg)

* Broker和集群

​	一个 Kafka 服务器也称为 **Broker**，它接受生产者发送的消息并存入磁盘；Broker 同时服务消费者拉取分区消息的请求，返回目前已经提交的消息。使用特定的机器硬件，一个 Broker 每秒可以处理成千上万的分区和百万量级的消息。

​	若干个 Broker 组成一个**集群（Cluster）**，其中集群内某个 Broker 会成为集群控制器（Cluster Controller），它负责管理集群，包括分配分区到 Broker、监控 Broker 故障等。在集群内，一个分区由一个 Broker 负责，这个 Broker 也称为这个分区的 Leader；当然一个分区可以被复制到多个 Broker 上来实现冗余，这样当存在 Broker 故障时可以将其分区重新分配到其他 Broker 来负责。下图是一个样例：

![image-20220222161459108](../img/image-20220222161459108.png)

​	Kafka 的一个关键性质是日志保留（retention），我们可以配置主题的消息保留策略，譬如只保留一段时间的日志或者只保留特定大小的日志。当超过这些限制时，老的消息会被删除。我们也可以针对某个主题单独设置消息过期策略，这样对于不同应用可以实现个性化。

#### 存储

​	**Kafka 的消息是存在于文件系统之上的**。Kafka 高度依赖文件系统来存储和缓存消息，一般的人认为 “磁盘是缓慢的”，所以对这样的设计持有怀疑态度。实际上，磁盘比人们预想的快很多也慢很多，这取决于它们如何被使用；一个好的磁盘结构设计可以使之跟网络速度一样快。

​	**Topic 其实是逻辑上的概念，面相消费者和生产者，物理上存储的其实是 Partition**，每一个 Partition 最终对应一个目录，里面存储所有的消息和索引文件。默认情况下，每一个 Topic 在创建时如果不指定 Partition 数量时只会创建 1 个 Partition。比如，我创建了一个 Topic 名字为 test ，没有指定 Partition 的数量，那么会默认创建一个 test-0 的文件夹，这里的命名规则是：`<topic_name>-<partition_id>`。

![img](../img/v2-346cb3219087ad26746e18f410954d9f_720w.jpg)

​	任何发布到 Partition 的消息都会被追加到 Partition 数据文件的尾部，这样的顺序写磁盘操作让 Kafka 的效率非常高（经验证，顺序写磁盘效率比随机写内存还要高，这是 Kafka 高吞吐率的一个很重要的保证）。

每一条消息被发送到 Broker 中，会根据 Partition 规则选择被存储到哪一个 Partition。如果 Partition 规则设置的合理，所有消息可以均匀分布到不同的 Partition中。

* 文件结构

​	假设我们现在 Kafka 集群只有一个 Broker，我们创建 2 个 Topic 名称分别为：「topic1」和「topic2」，Partition 数量分别为 1、2，那么我们的根目录下就会创建如下三个文件夹：

```text
    | --topic1-0
    | --topic2-0
    | --topic2-1
```

​	在 Kafka 的文件存储中，同一个 Topic 下有多个不同的 Partition，每个 Partition 都为一个目录，而每一个目录又被平均分配成多个大小相等的 **Segment File** 中，Segment File 又由 index file 和 data file 组成，他们总是成对出现，后缀 ".index" 和 ".log" 分表表示 Segment 索引文件和数据文件。

​	现在假设我们设置每个 Segment 大小为 500 MB，并启动生产者向 topic1 中写入大量数据，topic1-0 文件夹中就会产生类似如下的一些文件：

```text
    | --topic1-0
        | --00000000000000000000.index
        | --00000000000000000000.log
        | --00000000000000368769.index
        | --00000000000000368769.log
        | --00000000000000737337.index
        | --00000000000000737337.log
        | --00000000000001105814.index
        | --00000000000001105814.log
    | --topic2-0
    | --topic2-1
```

​	**Segment 是 Kafka 文件存储的最小单位。**Segment 文件命名规则：Partition 全局的第一个 Segment 从 0 开始，后续每个 Segment 文件名为上一个 Segment 文件最后一条消息的 offset 值。数值最大为 64 位 long 大小，19 位数字字符长度，没有数字用0填充。如 00000000000000368769.index 和 00000000000000368769.log。

以上面的一对 Segment File 为例，说明一下索引文件和数据文件对应关系：

<img src="../img/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3RyeWxs,size_16,color_FFFFFF,t_702.png" alt="img" style="zoom: 67%;" />

​	注意该 index 文件并不是从0开始，也不是每次递增1的，这是因为 Kafka 采取稀疏索引存储的方式，每隔一定字节的数据建立一条索引，它减少了索引文件大小，使得能够把 index 映射到内存，降低了查询时的磁盘 IO 开销，同时也并没有给查询带来太多的时间消耗。

​	因为其文件名为上一个 Segment 最后一条消息的 offset ，所以当需要查找一个指定 offset 的 message 时，通过在所有 segment 的文件名中进行二分查找就能找到它归属的 segment ，再在其 index 文件中找到其对应到文件上的物理位置，就能拿出该 message 。



#### 生产和消费	

* 发送
  * 保证吞吐量
    * 批量发送&消费
  * 局部顺序消息

![img](../img/v2-de25b264b72d1540b598c2362ddcc465_720w.jpg)

* 消费

  * partition分配

  ![img](../img/v2-3c4aef7b5d7f91ef1acaa7ec224a8272_720w.jpg)

​												![img](../img/v2-9c613777295125d63e23c91df4618cc9_720w.jpg)

![img](../img/v2-579ac0f4573a5060822e080e35a1f1c8_720w.jpg)

![img](../img/v2-f4bdede9192dce94c8b6f39c7f306ce6_720w.jpg)





#### 高可用

* 分区副本

![image-20220222161835697](../img/image-20220222161835697.png)

* 消息确认
  * 当消息写入所有in-sync状态的副本后，消息才会认为**已提交（committed）**。这里的写入有可能只是写入到文件系统的缓存，不一定刷新到磁盘。生产者可以等待不同时机的确认，比如等待分区主副本写入即返回，后者等待所有in-sync状态副本写入才返回。
  
    

### RocketMQ

#### 概念

  在rocketMq的中核心4组件为**namesrv**、**broker**、**consumer**、**producer**。

  **broker**：消息存储中心，主要用来存储消息并通过namesrv对外提供服务。

  **namesrv**：无状态的注册中心，功能用来保存broker的相关的元信息并提供给producer在发送消息过程中和提供给consumer消费消息过程中查找broker信息。

  **producer**：消息生产者，通过namesrv获取broker的地址并发送消息。

  **consumer**：消息消费者，通过namesrv获取broker的地址并消费消息。

  **queue**: Queue是组成Topic的更小单元，集群消费模式下一个消费者只消费该Topic中部分Queue中的消息

![查看源图像](../img/v2-612e7eaad6734189fdb6c927a9a1d520_r.jpg)

#### 存储

​	RocketMq的消息存储通过二级索引来进行，其中实际消息存储在Commit Log的逻辑队列中（磁盘文件消息顺序写），consume queue保存着每个消息消费队列的待消费的数据并且指向commit Log。

![img](../img/webp.png)

* 文件存储

![img](../img/webp3.png)



#### 生产和消费

* 生产
  * 自动创建
  * 发送队列负载均衡
* 消费
  * 消费模式 Pull Vs Push【长链接+PULL】
  * Rebalance

#### 高可用

* 主从节点备份

![img](../img/webp2.png)



* 同步刷盘&异步刷盘

![img](../img/rocketmq_design_2.png)

