---
layout: post
title: "minikube安装"
subtitle: "minikube国内版"
date: 2020-04-26
author: liying
category: kubernetes
tags: kubernetes
finished: true 
---
## 安装

安装教程：[阿里云维护的国内版minikube](https://yq.aliyun.com/articles/221687)

按步骤操作，下载驱动的时候三选一。默认是hyperkit，若本机已经安装有Docker，可以直接在start命令加上参数`--vm-drive=docker`指定驱动。

安装过程中出现镜像拉不下来的问题，一般是网络原因引起的。

```shell
output: [Unable to find image 'gcr.io/k8s-minikube/kicbase:v0.0.5@sha256:3ddd8461dfb5c3e452ccc44d87750b87a574ec23fc425da67dccc1f0c57d428a' locally docker: Error response from daemon: Get https://gcr.io/v2/: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers). See 'docker run --help'.] : exit status 125
```

内网无法访问gcr.io上的镜像，架梯子发现还是不行。需要设置http代理，如图。

![http-proxy](/img/http_proxy.png)

然后如下图即启动完成。

![done-minikube](/img/done_minikube.png)