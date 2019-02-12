---
title: 【spring-cloud】在网关中配置swagger
date: 2018-11-05 15:11:52
tags:
  - spring-cloud
---

## TODO LIST
- [1/2] 在你的服务中配置swagger接口的地址

在 application.yml 中配置

`springfox.documentation.swagger.v2.path: /{server path}/api-docs`

`{server path}` 为你的服务在网关中配置的上下文路径，比如下面的订单服务在网关的配置中，上下文路径为**order-server**：

```yaml
order:
  path: /order-server/**
  serviceId: oasis-ct-order-server
  stripPrefix: false
```

- [2/2] 在网关中配置接口访问URI

在 `application.yml` 中如下配置：
```yaml
swagger:
  apis:
    # 订单系统
    - name: order-server
      location: /order-server/api-docs
    # 支付系统
    - name: pay-server
      location: /pay/api-docs
``` 

## 原理

swagger的请求处理器 `Swagger2Controller` 中有个默认接口URI `DEFAULT_URL`，下面的注解用于替换默认URI：
```text
@PropertySourcedMapping(
      value = "${springfox.documentation.swagger.v2.path}",
      propertyKey = "springfox.documentation.swagger.v2.path")
```
具体该注解处理方法，见 `PropertySourcedRequestMappingHandlerMapping`