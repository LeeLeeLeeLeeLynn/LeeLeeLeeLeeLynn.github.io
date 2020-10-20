---

layout: post
title: "http请求因为content-length设置错误导致结果不正常"
subtitle: "Request method 'ET' not supported"
date: 2020-8-11
author: liying
category: http
tags: [http,content-length]
finished: true
---
## 遇到的问题

向接口发送请求时，出现`Request method 'ET' not supported`报错。

服务启动后的第一次请求不会报错，但是连续请求的第二次和之后的请求会报错，且接口为GET接口。

## 原因

第一次请求时，servlet截取到的HTTP请求为正确的'GET'。

由报错信息可以知道之后的请求中，后端servlet截取到的HTTP请求方法为‘ET’而不是‘GET’。

确认了发送的请求确实为GET后，思考为什么GET方法变成了ET，少了1位。

请求发送时，请求头中的因为直接拷贝的其他请求的请求头，content-length设置为了1。

而我们的请求为GET，并没有设置body。所以body为空。删除该header后，问题解决。

所以content-length长度一定要和body长度一致。

## content-length怎么影响读取http报文

当inputbuffer获取http报文时，根据content-length，知道总长度为header长度+content-length（即body长度）。

读取request和处理request由**Http11Processor**执行。处理逻辑如图所示。

![http11processor处理逻辑](../img/http11processor.png)

request请求在经过**Http11Processor**处理时，会首先调用**Http11InputBuffer**的方法对转换请求头、请求方法和请求协议。

所以在第一次请求时，解析没有问题，并且在处理完请求后返回执行结果。如果不是异步请求，在返回结果后，会调用 endRequest()来关闭当前request（关闭前响应已经返回）。

```java
if (!isAsync()) {
                // If this is an async request then the request ends when it has
                // been completed. The AsyncContext is responsible for calling
                // endRequest() in that case.
                endRequest();
            }
```

**Http11Processor **的endRequest()方法中又调用了**Http11InputBuffer**的endRequest()方法对当前请求做buffer相关的结束处理。

```java
void endRequest() throws IOException {

        if (swallowInput && (lastActiveFilter != -1)) {
            int extraBytes = (int) activeFilters[lastActiveFilter].end();
            byteBuffer.position(byteBuffer.position() - extraBytes);
        }
    }
```

endRequest()尝试去获取buffer中这次请求是否有需要读取的字节，用到的是**IdentityInputFilter**。在当前情况下，真正读取到的字节数比告知filter的应读字节数少了1（即content-length=1，而实际上不存在content）。

所以**IdentityInputFilter**中remaining=1>0，即**IdentityInputFilter**认为还需要读入1字节，所以会尝试去获取输入`int nread = buffer.doRead(this);`这是一个同步阻塞操作，直到读到下一个字节或超时。

```java
public long end() throws IOException {

        final boolean maxSwallowSizeExceeded = (maxSwallowSize > -1 && remaining > maxSwallowSize);
        long swallowed = 0;

        // Consume extra bytes.
        while (remaining > 0) {

            int nread = buffer.doRead(this);
            tempRead = null;
            if (nread > 0 ) {
                swallowed += nread;
                remaining = remaining - nread;
                if (maxSwallowSizeExceeded && swallowed > maxSwallowSize) {
                    // Note: We do not fail early so the client has a chance to
                    // read the response before the connection is closed. See:
                    // https://httpd.apache.org/docs/2.0/misc/fin_wait_2.html#appendix
                    throw new IOException(sm.getString("inputFilter.maxSwallow"));
                }
            } else { // errors are handled higher up.
                remaining = 0;
            }
        }
   // If too many bytes were read, return the amount.
        return -remaining;
}
```

那么如果服务器在socket超时关闭前收到了下一个请求，**IdentityInputFilter**将认为比预期读多了，返回多读的字节数，即 extra=读入字节数-remaining。

buffer的`position`会被设置为byteBuffer.position() - extraBytes。在我们的例子中，实际上下一次请求传入了306个byte，但是因为原先的remaining为1，所以**IdentityInputFilter**认为只有305个字符是属于下一次请求的。byteBuffer.position()是当前缓冲区的最后一个字符，位置为612，所以计算出下一次请求应该从612-305=307的地方开始读取。307的位置上存放的是‘E’，从这里开始少了第一位G。

![image-20201013152719434](/Users/liying/Library/Application Support/typora-user-images/image-20201013152719434.png)

处理完这些之后，**Http11Processor**会继续调用inputBuffer的nextRequest方法，这个方法为下一个请求进行buffer偏移量的调整。

```java
if (!isAsync() || getErrorState().isError()) {
                request.updateCounters();
                if (getErrorState().isIoAllowed()) {
                    inputBuffer.nextRequest();
                    outputBuffer.nextRequest();
                }
            }
```

方法中的这一段，会对将剩下未读出的字节复制到当前buffer的最前端，compact()方法**释放已读数据的空间，准备重新填充缓存区**。

再**调用flip()方法，切换为读就绪状态**

```java
if (byteBuffer.position() > 0) {
            if (byteBuffer.remaining() > 0) {
                // Copy leftover bytes to the beginning of the buffer
                byteBuffer.compact();
                byteBuffer.flip();
            } else {
                // Reset position and limit to 0
                byteBuffer.position(0).limit(0);
            }
        }
```

于是原先position=307-612这部分数据被复制到了0-305位置上。这个http请求被解析时，继续从positon=0开始读取并进行转换请求头等操作。以此往复，每一次请求都会少了报文的第一个字节。



参考：

https://www.cnblogs.com/chenpi/p/6475510.html

https://blog.fundebug.com/2019/09/10/understand-http-content-length/

