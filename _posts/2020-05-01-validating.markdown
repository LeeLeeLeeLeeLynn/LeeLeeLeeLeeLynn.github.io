---
layout: post
title: "error: error validating "nginx.yaml": error validating data: the server could not find the requested resource; if you choose to ignore these errors, turn validation off with --validate=false"
subtitle: "kubectl create 报错：“error validating data”"
date: 2020-05-01
author: liying
category: Kubernetes
tags: kubernetes
finished: true 
---
## 问题&解决

如下简单的yaml用于创建一个可用shell与nginx容器交互的pod。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  shareProcessNamespace: true
  containers:
  - name: nginx
    image: nginx
  - name: shell
    image: busybox
    stdin: true
    tty: true

```

使用`kubectl create -f nginx.yml`命令创建pod。

出现如下报错。

```shell
error: error validating "nginx.yaml": error validating data: the server could not find the requested resource; if you choose to ignore these errors, turn validation off with --validate=false

```

看起来像是验证数据失败错误，但是没有具体提示。

创建命令后增加`--validate=false`参数可以创建pod，但是之后执行`kubectl attach -it nginx -c shell`连接到shell容器的tty上仍然报错。

```shell
 error: unable to upgrade connection: container shell not found in pod nginx_default
```

说明pod内部仍然没有创建成功shell容器。

github上有issue提到类似的问题：https://github.com/rancher/rancher/issues/20606

![issue](/img/issue.png)

解决方案是更新kubectl。

本机系统是macOS，安装时使用homebrew直接安装的，使用`kubectl version `检查一下client版本是1.6.6，并且是最新版。

```shell
Client Version: version.Info{Major:"1", Minor:"6+", GitVersion:"v1.6.6-dirty", GitCommit:"7fa1c1756d8bc963f1a389f4a6937dc71f08ada2", GitTreeState:"dirty", BuildDate:"2020-04-25T03:23:53Z", GoVersion:"go1.8.3", Compiler:"gc", Platform:"darwin/amd64"}
Server Version: version.Info{Major:"1", Minor:"17", GitVersion:"v1.17.3", GitCommit:"06ad960bfd03b39c8310aaf92d1e7c12ce618213", GitTreeState:"clean", BuildDate:"2020-02-11T18:07:13Z", GoVersion:"go1.13.6", Compiler:"gc", Platform:"linux/amd64"}

```

包管理系统没有把client升级到最新版。

于是使用[kubernetes提供的curl安装方法]([https://kubernetes.io/zh/docs/tasks/tools/install-kubectl/#%e9%80%9a%e8%bf%87-curl-%e5%91%bd%e4%bb%a4%e5%ae%89%e8%a3%85-kubectl-%e5%8f%af%e6%89%a7%e8%a1%8c%e6%96%87%e4%bb%b6](https://kubernetes.io/zh/docs/tasks/tools/install-kubectl/#通过-curl-命令安装-kubectl-可执行文件))。

再看一下version是1.18.2了。

```
Client Version: version.Info{Major:"1", Minor:"18", GitVersion:"v1.18.2", GitCommit:"52c56ce7a8272c798dbc29846288d7cd9fbae032", GitTreeState:"clean", BuildDate:"2020-04-16T11:56:40Z", GoVersion:"go1.13.9", Compiler:"gc", Platform:"darwin/amd64"}
Server Version: version.Info{Major:"1", Minor:"17", GitVersion:"v1.17.3", GitCommit:"06ad960bfd03b39c8310aaf92d1e7c12ce618213", GitTreeState:"clean", BuildDate:"2020-02-11T18:07:13Z", GoVersion:"go1.13.6", Compiler:"gc", Platform:"linux/amd64"}
```

重新执行create语句，创建成功。