---
layout: post
title: "docker容器"
subtitle: "基于nameSpace和Cgroup实现隔离和限制"
date: 2019-12-11
author: liying
category: docker
tags: docker
finished: true
---

*“一个正在运行的Docker容器，其实就是一个启用了多个**Linux Namespace**的应用进程，而这个进程能够使用的资源量，则受**Cgroups**配置的限制。”*

## NameSpace ：是Linux在创建新进程的一个可选参数

- NameSpace可以让进程只看到与自己相关的资源，对其他nameSpace下的进程及其资源无感

- nameSpace的隔离能力：
  ![ability](/img/nameSpace.png)

- 相关函数：
  - `clone()`  创建新进程的时候同时创建namespace 
  
  - `setns()` 将当前进程加入已有的namespace
  
    - ```bash
      int setns(int fd, int nstype)；
      ```
  
      fd为文件描述符，指向对应`/proc/[pid]/ns/`中的一个namesapace。
  
      nstype为namespace的类型，对fd进行检查，判断类型是否符合，若不进行检查则填写0。
  
      nstype取值见：http://man7.org/linux/man-pages/man2/setns.2.html
  
    【nameSpace是对各种资源分别进行隔离，而不是一键隔离所以每种资源对应一个id，指向相同NameSpace id的资源共享，对其余则不可见】
  
  - `unshare()` 在原进程的基础上，创建并加入新的namespace

参考：[https://www.cnblogs.com/sparkdev/p/9365405.html](nameSpace)

## Cgroup：限制一个进程组能够使用的资源上限

cgroup通过文件的方式暴露给用户

`/sys/fs/cgroup/[子系统]/[控制组名]/[具体资源种类]` 

- [子系统] cgroup下有多个子系统，如cpuset、cpu、memory等。是可以被cgroup进行限制的资源种类。

  - `cat /etc/cgconfig.conf` 查看子系统挂载目录【注：目录不存在的话，需要挂载cgroup（详情谷歌）】

    例, `cpu = /sys/fs/cgroup/cpu;`

- [控制组名]在子系统下创建一个目录，即控制组。操作系统则会在新创建的控制组下，自动生成多个具体资源文件。

- [具体资源种类]通过echo方法将限制写入具体资源种类文件中，即可以达到对于该控制组下的某个具体资源种类进行限制的目的。

## change root：切换根目录

*“Mount Namespace 修改的，是容器进程对文件系统“挂载点”的认知。”*

即Mount Namespace需要进行挂载操作后才能生效。

所以docker在容器启动前，挂载根目录，使得对宿主机其他文件不可见。

linux提供`chroot`命令来改变进程根目录，而docker会有优先使用`pivot_root`系统调用，若不支持才用`chroot`。原因是：

- chroot：改变某个进程或线程的根目录
- pivot_root：改变整个系统的根目录，并且去掉对之前rootfs的依赖，以便unmount。
- 参考：[chroot，pivot_root和switch_root 区别](https://blog.csdn.net/u012385733/article/details/102565591)

而这个挂载在容器根目录上、用来为容器进程提供隔离后执行环境的文件系统，叫做 [**rootfs**](https://leeleeleeleelynn.github.io/docker/contain.html)。

## 总结：

因此容器仍然还是宿主机上的**普通进程**，共享一个宿主机的内核，运行在同一个宿主机的操作系统上，只是使用了**namespace**及**cgroup**进行了隔离和限制。
<u>这意味着，如果要在Windows宿主机上运行Linux容器，或者在低版本的Linux宿主机上运行高版本的Linux容器，都是行不通的。</u>

优点：“敏捷”和“高性能”，与vm相比，不存在因虚拟化带来的性能损耗。

缺点：隔离不彻底。例：top出来的cpu和内存仍然是宿主机的信息。

所以Docker的容器创建步骤是：

1. 使用namespace进行资源隔离。
2. 通过cgroup对资源进行限制。
3. 切换根目录，根目录中是一个操作系统所包含的文件、配置和目录。（**不包含内核**）

![docker结构视图](/img/docker_core.png)

