---
layout: post
title: "docker容器"
subtitle: "基于nameSpace和Cgroup实现隔离和限制"
date: 2019-12-11
author: liying
category: docker
tags: docker
finished: false
---

## 二、容器既然是一个封闭的进程，那么外接程序是如何进入容器这个进程的呢

cgroup通过文件的方式暴露给用户

`/sys/fs/cgroup/[子系统]/[控制组名]/[具体资源种类]` 

- [子系统] cgroup下有多个子系统，如cpuset、cpu、memory等。是可以被cgroup进行限制的资源种类。

  - `cat /etc/cgconfig.conf` 查看子系统挂载目录【注：目录不存在的话，需要挂载cgroup（详情谷歌）】

    例, `cpu = /sys/fs/cgroup/cpu;`

- [控制组名]在子系统下创建一个目录，即控制组。操作系统则会在新创建的控制组下，自动生成多个具体资源文件。

- [具体资源种类]通过echo方法将限制写入具体资源种类文件中，即可以达到对于该控制组下的某个具体资源种类进行限制的目的。

## 三、docker commit对挂载点volume内容修改的影响是什么？

## 四、容器与宿主机如何进行文件读写？或volume是为了解决什么题？

## 五、Docker的copyData功能是什么？解决了什么问题？

## 六、bind mount机制是什么？

## 七、cgroup Namespace的作用是什么？



## 总结：

因此容器仍然还是宿主机上的**普通进程**，共享一个宿主机的内核，运行在同一个宿主机的操作系统上，只是使用了**namespace**及**cgroup**进行了隔离和限制。
<u>这意味着，如果要在Windows宿主机上运行Linux容器，或者在低版本的Linux宿主机上运行高版本的Linux容器，都是行不通的。</u>

优点：“敏捷”和“高性能”，与vm相比，不存在因虚拟化带来的性能损耗。

缺点：隔离不彻底。例：top出来的cpu和内存仍然是宿主机的信息。