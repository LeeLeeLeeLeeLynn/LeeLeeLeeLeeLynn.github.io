---
layout: post
title: "@SneakyThrows"
subtitle: "使用@SneakyThrows来隐藏异常"
date: 2020-10-20
author: liying
category: java
tags: [java,注解]
finished: true

---

## @SneakyThrows

SneakyThrows用于隐藏CheckedException。

Java异常分为两种，

1.RuntimeException：运行时异常

2.Exception：CheckedException，必须被抛出或捕获。

两种情况下我们不想处理或抛出异常：

1. 处理不了的异常，直接抛出。通常的做法是包一层RuntimeException

   ```java
   try{...
   }catch(Exception e){
   	throw new Runnable(e);
   }
   ```

   但是这样的操作会可能会遮掩住引起异常的真正原因。

2. 抛出一个不可能出现的异常，例如，`new String(someByteArray, "UTF-8");`声明可以抛出一个，`UnsupportedEncodingException`但是根据JVM规范，UTF-8*必须*始终可用。

## 使用@SneakyThrows偷偷抛出受检异常

使用@SneakyThrows注释方法，可以注明异常类型或不注明。

```java
import lombok.SneakyThrows;

public class SneakyThrowsExample implements Runnable {
  @SneakyThrows(UnsupportedEncodingException.class)
  public String utf8ToString(byte[] bytes) {
    return new String(bytes, "UTF-8");
  }
  
  @SneakyThrows
  public void run() {
    throw new Throwable();
  }
}
```

Lombok填充的代码，若注明了异常类型，则生成捕获该异常的catch块，若未注明，则catch父类Throwable。

```java
import lombok.Lombok;

public class SneakyThrowsExample implements Runnable {
  public String utf8ToString(byte[] bytes) {
    try {
      return new String(bytes, "UTF-8");
    } catch (UnsupportedEncodingException e) {
      throw Lombok.sneakyThrow(e);
    }
  }
  
  public void run() {
    try {
      throw new Throwable();
    } catch (Throwable t) {
      throw Lombok.sneakyThrow(t);
    }
  }
}
```

## Lombok的实现方法

Lombok没有将CheckedException包装成一个新的RuntimeException，而是使用泛型将Throwable强制转换成了RuntimeException，从而骗过了编译器。

```java
    public static RuntimeException sneakyThrow(Throwable t) {
        if (t == null) {
            throw new NullPointerException("t");
        } else {
            return (RuntimeException)sneakyThrow0(t);
        }
    }

    private static <T extends Throwable> T sneakyThrow0(Throwable t) throws T {
        throw t;
    }
```

JVM并不关心异常是CheckedException还是RuntimeException，所以事实上我们的CheckedException没有被重新包装，没有被忽略或重定义。

参考：

1. https://www.jianshu.com/p/7d0ed3aef34b

2. https://projectlombok.org/features/SneakyThrows

