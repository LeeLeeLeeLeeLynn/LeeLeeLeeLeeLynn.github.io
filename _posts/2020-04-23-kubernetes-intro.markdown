---
layout: post
title: "kubernetes基础知识"
subtitle: "基于Kubernetes第9讲"
date: 2020-04-23
author: liying
category: kubernetes
tags: kubernetes
finished: false
---

## Kubernetes 架构设计

kubernetes采用了master（控制节点）+node（计算节点）的节点的结构。

### Master节点

Master节点由三部分组成：

#### Controller Manager

负责容器编排

#### API Server

负责API服务

#### Scheduler

负责调度

![kubernetes架构](https://static001.geekbang.org/resource/image/8e/67/8ee9f2fa987eccb490cfaa91c6484f67.png)

### Node节点

**Node**节点则以**kubelet**为核心。kubelet中的agent与master通信。

*“在 Kubernetes 项目中，**kubelet **主要负责同容器运行时（比如 Docker 项目）打交道。”*

如上图，kubelet与多个组件进行交互。

#### kubelet------->容器运行时：通过CRI（Container Runtime Interface）

CRI定义了容器运行时的各项核心操作。

#### 容器运行时------->Linux操作系统：通过OCI（Open Container Initiative）

OCI对于容器的创建、删除、查看等操作进行了定义。OCI与底层Linux交互。所以kubelet不关心具体容器。

#### kubelet------->Device Plugin（管理 GPU 等宿主机物理设备的主要组件）：通过grpc调用

#### kubelet------->Volume Plugin（存储插件）:通过CSI（Container Storage Interface）

为容器配置持久化存储

#### kubelet------->Networking（网络插件）:通过CNI（Container Network Interface）

为容器配置网络

## Kubernetes核心功能

![Kubernetes核心功能](https://static001.geekbang.org/resource/image/16/06/16c095d6efb8d8c226ad9b098689f306.png)

#### Pod

Pod是最小部署单元，由一组容器组成，一个Pod中的容器共享所有资源。

#### Service

service是通过apiserver创建出来的对象实例，作为 Pod 的代理入口（Portal），从而代替 Pod 对外暴露一个固定的网络地址。