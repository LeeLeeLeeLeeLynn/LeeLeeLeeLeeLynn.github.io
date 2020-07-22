---
layout: post
title: "final变量还可以被修改吗？"
subtitle: "final可以修饰类、方法和变量。修饰引用变量时，不允许变量进行重定向。"
date: 2020-07-17
author: liying
category: java基础
tags: java
finished: true 
---
## final关键字

通常认知中，final 修饰的变量一旦初始化，就不能被修改。

在org.apache.curator.framework.recipes.locks.InterProcessMutex源码中遇到了final变量属性被修改的情况。

LockData是一个内部类，获取锁时，从map中获取了lockData，并对其中的lockCount属性进行原子自增操作。

```java
if (lockData != null) {
            lockData.lockCount.incrementAndGet();
            return true;
}
```

```java
 private static class LockData {
        final Thread owningThread;
        final String lockPath;
        final AtomicInteger lockCount;

        private LockData(Thread owningThread, String lockPath) {
            this.lockCount = new AtomicInteger(1);
            this.owningThread = owningThread;
            this.lockPath = lockPath;
        }
    }
```

通常对final属性进行修改时，无法通过编译，

如：

```java
 final int b=1;
 //报错
 //b++; 
 final Integer c=new Integer(1);
 //报错
 //c++; 
```

但是如果final修饰的是对象，可以修改内部变量，

如：

```java
 final AtomicInteger a = new AtomicInteger(1);
 a.incrementAndGet();
 final List<Integer> d=new ArrayList<>();
 d.add(1);
```

这是由于final修饰对象时，引用变量指向的是实际的对象，但其存储的是所指向对象的地址，所以值不能改变，不意味着所指的对象属性不能改变。

## final修饰符使用【复习】

#### 修饰类

类不能被继承，无法拥有自己的子类。

#### 修饰方法

方法不能被重写

#### 修饰变量

- 基本数据类型，不能修改。
- 引用类型，不能重新赋值对象，但是对象中的变量可以修改。

