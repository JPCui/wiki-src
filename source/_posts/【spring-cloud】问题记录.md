---
title: 【spring-cloud】问题记录
date: 2018-11-05 15:11:52
tags:
  - spring-cloud
---

- feign 请求 get 后返回 post 404 error

  - 检查请求路径是否正确
  - 检查是否Get请求参数为对象，@see [feign not support object param by GET](feign%20not%20support%20object%20param%20by%20GET.md)
  
- feign 参数中有 Date 参数时，保证客户端和服务端格式一样

  spring boot 下配置：
  ```
  spring:
    jackson:
      date-format: yyyy-MM-dd HH:mm:ss
  ```
  
- @FeignClient 被注册为 RequestHandler，导致与服务接口 duplicate

> @FeignClient with top level @RequestMapping annotation is also registered as Spring MVC handler

see: https://github.com/spring-cloud/spring-cloud-netflix/issues/466