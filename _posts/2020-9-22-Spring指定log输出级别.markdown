---
layout: post
title: "单独指定某个Logger的输出级别"
subtitle: "某个Logger输入的大小过大，想要不打印这部分日志怎么办？"
date: 2020-9-20
author: liying
category: http
tags: [java,DelayQueue]
finished: true
---

## Spring配置调整

调整某个类的日志输出级别，在properties中增加配置：

```properties
logging.level.com.hellobike.tesla.utils.http.OkHttpHelper=ERROR
```

在log4j.properties中修改logger的配置

```properties
log4j.rootLogger = DEBUG, FILE
....
log4j.appender.FILE.Threshold = DEBUG
....
```

在JVM Options 指定：

```java
-Dlogging.level=DEBUG
```



## slf4j和log4j

通过idea的dependency梳理，可以看到引入了三个包，分别是log4j，slf4j-api和slf4j-log4j12。

![image-20210104200004554](../img/image-20210104200004554.png)

* slf4j-api是一套抽象日志接口，只包含接口而没有实现。
* log4j是日志接口实现。
* slf4j-log4j12使得log4j对接到slf4j，使用slf4j的接口而实际调用log4j的实现。

<img src="/Users/liying/Library/Application Support/typora-user-images/image-20210104201225232.png" alt="image-20210104201225232" style="zoom:50%;" />

图源：

注：log4j可以单独使用，而不是用slf4j



## 为什么要使用slf4j

日志的实现有很多套，例如：log4j、logback和java.util.logging等。

基于面向接口编程的思想，我们习惯调用接口，以便于灵活地变更实现。

slf4j也是一样，使用了门面模式(Facade Pattern)，开发者无须关心具体实现，adaptation连通了接口和实现。如果有一天需要变更日志实现时，只需要替换依赖的jar包，及相关配置文件就能完成，否则，需要修改大量的代码。而且，举例来说，如果引入的基础平台的包使用了logback的日志系统，而当前应用使用的是log4j的日志系统，那么需要维护两套日志系统。