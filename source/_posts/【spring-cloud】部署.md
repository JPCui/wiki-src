---
title: 【spring-cloud】部署
date: 2018-11-05 15:11:52
tags:
  - spring-cloud
---

## 构建docker容器

[如何构建容器](../docker/how-to-build-container.md)

## 优雅关闭服务

[优雅关闭服务](/spring-boot/优雅关闭服务.md)

## 远程调试

修改完成后，请修改文档：[基础服务列表](../oasis-ct/基础服务列表.md)

启动时，加上 `-Xdebug -Xrunjdwp:transport=dt_socket,server=y,suspend=n,address=[debug port]`

注意加上远程调试前，也要在docker中配置调试端口的**映射**

下面是一个完整的配置:

> java 
> -Dfile.encoding=UTF-8 
> -Duser.timezone=GMT+08 
> -Dspring.profiles.active=[profile]
> -Xdebug
> -Xrunjdwp:transport=dt_socket,server=y,suspend=n,address=[debug port]
> -jar [jar]


