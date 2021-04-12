---
layout: post
title: "链接资源未释放导致的请求链接池超时？"
subtitle: "Timeout waiting for connection from pool？"
date: 2021-03-31
author: liying
category: java
tags: [java,httpClient]
finished: false


---

* 目录
{:toc #markdown-toc}


## 问题

使用HttpClient 调用http接口，调用多次后出现

```
java org.apache.http.conn.ConnectionPoolTimeoutException: Timeout waiting for connection from pool
```

调用代码

```java
    /**
     * 调用接口1
     */
		public List<String> getRunningConsumers(String ip) throws IOException {
        HttpResponse response = HttpClientUtil.doGet("http://" + ip + ":9889", "/getRunningConsumers", new HashMap<>());
        String body = new BufferedReader(new InputStreamReader(response.getEntity().getContent()))
                .lines().collect(Collectors.joining(System.lineSeparator()));
        List<String> result = JsonUtils.parseArray(body, String.class);
        return result;
    }
    /**
     * 调用接口2
     */
    public List<Task> getRunningTasks(String ip) throws IOException {
        HttpResponse response = HttpClientUtil.doGet("http://" + ip + ":9889", "/getRunningTasks", new HashMap<>());
        String body = new BufferedReader(new InputStreamReader(response.getEntity().getContent())).lines().collect(Collectors.joining());
        List<Task> result = JsonUtils.parseArray(body, Task.class);
        return result;
    }
```

然而调用接口1时不会出现该问题，调用接口2时，必现。



## 检索原因

网络检索到的原因以及处理方式基本为两种

1. 链接池过小：修改连接池配置，增加连接数

   ```java
   RequestConfig config = RequestConfig.custom()  
       .setSocketTimeout(socketTimeout)  
       .setConnectTimeout(connectTimeout)  
       .setConnectionRequestTimeout(connectionRequestTimeout).build();  
   httpClient = HttpClients.custom().setDefaultRequestConfig(config)  
        .setMaxConnTotal(maxConnTotal)  //增加最大链接数，支持更多链接
        .setMaxConnPerRoute(maxConnPerRoute).build(); 
   ```

   - 应对场景：高并发场景下，短时间偶发的链接数不够用。
   - 参考：[HttpClient 并发报 Timeout waiting for connection from pool](https://www.izhizheng.org/post/HttpClient-Concurrent-Timeout/)】

2. 连接未释放：调用`HttpResponse.getEntity()`后，会自动release连接；或手动释放。



## 调试问题

1. 确认不是短时间高并发
2. 已经调用了`HttpResponse.getEntity()`
3. 接口1返回参数为`List<String>`，状态为释放；接口2返回参数为`List<Object>`，状态为未释放。
4. 单步调试发现问题在于getEntity()读取流的方式。



## 接口1 VS 接口2

为什么接口1可以关闭连接而接口2不行呢？

堆栈信息如下：

接口1:

![image-20210407200331004](../img/image-20210407200331004.png)



接口2:

![image-20210407200409142](../img/image-20210407200409142.png)

对比发现接口1getEntity()返回的entity是`BasicHttpEntity`，而接口2返回的getEntity()返回的Entity类是`DecompressingEntity`。

都是用了StreamDecoder进行解码，都调用EofSensorInputStream的read()方法，read()方法每次读写都会校验EOF，然而接口2的方法，不会读取到offset=-1，只会到0。【原因之后在研究】

如果ResponseEntityProxy检测到eof后，会releaseConnection()，所以解决方案2中的，getEntity会释放连接。

ResponseEntityProxy源码如下：

```java
    @Override
    public boolean eofDetected(final InputStream wrapped) throws IOException {
        try {
            // there may be some cleanup required, such as
            // reading trailers after the response body:
            if (wrapped != null) {
                wrapped.close();
            }
            releaseConnection();
        } catch (final IOException ex) {
            abortConnection();
            throw ex;
        } catch (final RuntimeException ex) {
            abortConnection();
            throw ex;
        } finally {
            cleanup();
        }
        return false;
    }
```



## 解决方案

增加：

```java
//手动关闭连接
response.getEntity().getContent().close();
```

