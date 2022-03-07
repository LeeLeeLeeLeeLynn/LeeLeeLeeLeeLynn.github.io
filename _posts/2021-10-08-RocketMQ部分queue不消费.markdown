```
layout: post
title: "消息队列 | RocketMq某些队列阻塞不消费"
subtitle: "RocketMQ部分queue显示有积压，位点保持在一个位置，没有更新。[转载]"
date: 2021-10-08
author: liying
category: RocketMQ
tags: [RocketMQ,java]
finished: true
```

* 目录
{:toc #markdown-toc}


公司小伙伴反馈自己负责的RocketMq集群忽然有两个队列不消费了，消息堆积达到了1万多条，这个肯定不正常。

以下是当时的消费组的实际消费情况 ![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fc3ceec73e524811938008717332b290~tplv-k3u1fbpfcp-watermark.awebp)

从上面的图中可以看出来，有两个队列严重阻塞了，好久没有上报过offset了。

通过以往自己的经验，一般来说，RocketMq部分队列消费失败，主要有以下三个原因

1. 消费者的集群数量有问题，比如之前笔者有写过一篇文章，也是介绍RocketMq部分队列不能正常消费的，有兴趣的可以看看[RocketMq 部分队列不能消费问题排查](https://link.juejin.cn?target=https%3A%2F%2Fwww.shared-code.com%2Farticle%2F139)
2. 消费者的consumer group乱用，就是明明你不消费这个topic , 但是你的消费组名称取的跟别人正常消费的一样，这个时候，就会导致一部分队列分配到你这个节点身上，但是这些节点你本身又没有去消费，就会造成消费混乱
3. 消费者线程阻塞

上面列的三点，基本上罗列了笔者遇到的情况以及能想到的情况，欢迎大家补充！

通过逐一排查，第一点和第二点的可能性几乎没有，大家用的都还算比较规范，那就怀疑第三点了。

判断一个线程是否阻塞，最好的办法就是通过`jstack`命令查看整个线程栈的信息。

这里笔者使用 [gceasy.io/](https://link.juejin.cn?target=https%3A%2F%2Fgceasy.io%2F) 来做线程分析

第一次拉取

![img](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/80a4986a4d224ed7b55fe49c0e040d19~tplv-k3u1fbpfcp-watermark.awebp)

第二次拉取

![img](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c12e85d3e7ca4914a100476a3f3d65d9~tplv-k3u1fbpfcp-watermark.awebp)

拉了两次线程栈信息，清楚的可以看到两次都有5个rocketMQ消费端线程处于运行状态， 这个就有很大的问题。接下来看下栈执行的代码，

```log
ConsumeMessageThread_20
priority:5 - threadId:0x00007fcbbdca7000 - nativeId:0x3675 - nativeId (decimal):13941 - state:RUNNABLE
stackTrace:
java.lang.Thread.State: RUNNABLE
at java.net.SocketInputStream.socketRead0(Native Method)
at java.net.SocketInputStream.socketRead(SocketInputStream.java:116)
at java.net.SocketInputStream.read(SocketInputStream.java:171)
at java.net.SocketInputStream.read(SocketInputStream.java:141)
at sun.security.ssl.InputRecord.readFully(InputRecord.java:465)
at sun.security.ssl.InputRecord.readV3Record(InputRecord.java:593)
at sun.security.ssl.InputRecord.read(InputRecord.java:532)
at sun.security.ssl.SSLSocketImpl.readRecord(SSLSocketImpl.java:975)
- locked <0x00000000cceb4ab8> (a java.lang.Object)
at sun.security.ssl.SSLSocketImpl.readDataRecord(SSLSocketImpl.java:933)
at sun.security.ssl.AppInputStream.read(AppInputStream.java:105)
- locked <0x00000000cceb6098> (a sun.security.ssl.AppInputStream)
at java.io.BufferedInputStream.read1(BufferedInputStream.java:284)
at java.io.BufferedInputStream.read(BufferedInputStream.java:345)
- locked <0x00000000cceb6070> (a java.io.BufferedInputStream)
at sun.net.www.MeteredStream.read(MeteredStream.java:134)
- locked <0x00000000ccecbe30> (a sun.net.www.MeteredStream)
at java.io.FilterInputStream.read(FilterInputStream.java:133)
at sun.net.www.protocol.http.HttpURLConnection$HttpInputStream.read(HttpURLConnection.java:3454)
at sun.nio.cs.StreamDecoder.readBytes(StreamDecoder.java:284)
at sun.nio.cs.StreamDecoder.implRead(StreamDecoder.java:326)
at sun.nio.cs.StreamDecoder.read(StreamDecoder.java:178)
- locked <0x00000000ccecab68> (a java.io.InputStreamReader)
at java.io.InputStreamReader.read(InputStreamReader.java:184)
at java.io.BufferedReader.fill(BufferedReader.java:161)
at java.io.BufferedReader.readLine(BufferedReader.java:324)
- locked <0x00000000ccecab68> (a java.io.InputStreamReader)
at java.io.BufferedReader.readLine(BufferedReader.java:389)
at com.xxx.utils.GaodeUtil.readFileByUrl(GaodeUtil.java:87)
at com.xxx.utils.GaodeUtil.getDirection(GaodeUtil.java:73)
at com.xxx.utils.GaodeUtil.getDistanceByLngLat(GaodeUtil.java:198)
at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:511)
at java.util.concurrent.FutureTask.run(FutureTask.java:266)
at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
at java.lang.Thread.run(Thread.java:748)
复制代码
```

5个线程的栈信息几乎一致，可以看到都是堵在了 `GaodeUtil.readFileByUrl` 这个方法上面，最终底层是堵在了socketRead上面。

找到了业务代码就很好办了，直接去代码里面查看代码。

```java
private String readFileByUrl(String urlString) {
        try {
            StringBuffer html = new StringBuffer();
            URL url = new URL(urlString);
            HttpURLConnection conn = (HttpURLConnection)url.openConnection();
            InputStreamReader isr = new InputStreamReader(conn.getInputStream(), "UTF-8");
            BufferedReader br = new BufferedReader(isr);
            String temp;
            while ((temp = br.readLine()) != null) {
                html.append(temp).append("\n");
            }
            br.close();
            isr.close();
            return html.toString();
        } catch (Exception e) {
            log.error("GaodeUtil.readFileByUrl error:", e);
            return null;
        }
    }
复制代码
```

`readFileByUrl` 的代码如上， 这帮兔崽子写代码都不仔细思考的，这个很明显可以看出来问题，在发起http连接的时候，没有设置超时时间。

> HttpURLConnection是基于HTTP协议的，其底层通过socket通信实现。如果不设置超时（timeout），在网络异常的情况下，可能会导致程序僵死而不继续往下执行

好了，问题查清楚了， 是由于消费消息处理业务的时候，发起http请求，没有设置超时时间，导致在出现网络异常的情况下，程序直接僵死。

解决办法就是 重启机器（恢复业务运行）， 修改代码，增加超时时间。



作者：sharedCode
链接：https://juejin.cn/post/6901874475248140295
来源：稀土掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。



