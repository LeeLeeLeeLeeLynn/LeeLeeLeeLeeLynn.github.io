---
layout: post
title: "maven加载失败"
subtitle: "公司内部使用私有maven仓库，加载maven时出错，需要修改setting文件，所以记录一下setting.xml默认位置"
date: 2020-07-13
author: liying
category: environment
tags: [environment,MacOS,maven]
finished: true 
---
## maven 的setting.xml文件 

首次加载maven依赖时飘红一般都是因为配置问题。需要修改setting.xml文件。

全局的在: `${M2_HOME}/conf/settings.xml`

用户的需要自己配置，可以复制一份全局的放在下面的用户目录下【一般不推荐修改全局配置】

用户路径（默认）:`～/.m2/setting.xml`

用户配置大于全局配置。

在idea中可以对maven的setting文件进行配置，使用配置好的setting文件。![maven-setting](../img/maven-setting.png)

在idea中可以override用户配置文件setting.xml的目录，直接使用图形界面可以拉到正确的依赖。

但是在终端中使用mvn命令构建，maven仍然会从默认的位置读取setting文件，若setting文件不存在，则会出现问题。若要使用指定的setting文件来拉去依赖，则需要使用如下命令：

```shell
mvn --settings YourOwnSettings.xml clean install
```

或者

```shell
mvn -s YourOwnSettings.xml clean install
```

