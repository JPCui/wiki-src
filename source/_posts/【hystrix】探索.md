---
title: 【hystrix】探索
date: 2018-11-05 15:11:52
tags:
  - 
---

# 断路器

作用: 防止一个服务接口错误可能会导致级联故障. 

触发条件: 当**一个周期**内, 单个接口调用次数超过**请求调用阈值**, 或失败率达到**失败百分比阈值**,
断路器将会打开, 并且请求会失败.

降级: 为了防止失败和断路, 开发中可以提供一个回调函数.
回调函数可以是另一个受Hystrix保护的接口, 静态数据或空值. 

![](https://raw.githubusercontent.com/spring-cloud/spring-cloud-netflix/1.4.x/docs/src/main/asciidoc/images/HystrixFallback.png)


## 使用方式

引入依赖:
```
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
```

启用:

`@EnableCircuitBreaker`

See [here](https://github.com/Netflix/Hystrix/tree/master/hystrix-contrib/hystrix-javanica#configuration) for more.

See the [Hystrix wiki](https://github.com/Netflix/Hystrix/wiki/Configuration) for details on the properties available.





