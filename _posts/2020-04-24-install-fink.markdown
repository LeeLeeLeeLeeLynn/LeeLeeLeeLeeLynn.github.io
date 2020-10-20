---
layout: post
title: "安装fink在mac上使用apt-get"
subtitle: "fink安装"
date: 2020-04-24
author: liying
category: mac
tags: [fink,mac]
finished: true
---
## 在mac上使用apt-get

在linux上使用apt-get进行包管理很方便，但是在mac上不能直接使用apt-get。两种解决方案：

- 使用mac-os支持的homebrew
- 使用fink

> Fink 项目希望把 Unix 上各种开放源码软件带到 Darwin 和 Mac OS X 平台上。 我们通过修改 Unix 软件使得它可以在 Mac OS X 上编译和运行（“移植”）,并提供一个方便的分发系统使得每个人都可以下载和使用它。 Fink 使用 Debian 中的象 dpkg 和 apt-get 等工具来提供强大的二进制软件包管理。

### fink安装

[官网下载地址](http://www.finkproject.org/download/srcdist.php)

![fink-download](/img/fink_download.png)

**方法一**: （macOS 10.9~10.15适用）

1. 点击helper script下载安装脚本
2. 解压后运行Install Fink.tool
3. 提示安装Xcode，只用Xcode命令行的话可以选N，不安装。
4. 然后会自动开始下载XQuartz和fink包，根据提示操作。

**方法二**：

由于方法一下载XQuartz会比较慢，所以[XQuartz和Fink下载](https://pan.baidu.com/s/1bh4yDo#list/path=%2F)（来源博客：https://blog.csdn.net/leilba/article/details/50753871）

但是博主提供的Fink不支持我现在的10.14,所以

1. 安装上文提供的Xquartz.Dmg。

2. 从官网下载页下载fink-0.45.1.tar.gz。

3. 进入目录使用`./bootstrap`开始下载安装（速度比较慢），一路默认选项即可。

4. 安装完成后

   ```shell
   #set up this terminal session environment to use Fink
   . /sw/bin/init.sh
   #add '. /sw/bin/init.sh' to the init script '.profile' or'.bash_profile' in your home directory
   /sw/bin/pathsetup.sh
   ```

   多次安装失败则默认/sw路径会变成sw2、sw3等，则上述命令需要将/sw进行替换。

   提示”OK“。即可以使用apt-get了。

