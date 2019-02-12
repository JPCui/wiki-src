---
title: 【项目管理】spring-cloud接口定义规范
date: 2018-11-05 15:11:52
tags:
  - 
---

接口分两种, 一种是服务间调用, 一种是前端调用

建议 server 接口放在 controller.server 包下面

server 接口禁止外网访问,所以访问路径最好区分开放接口访问路径

## 调用者配置

```yaml
server:
  [server]:
    # 服务的serviceId
    serviceId: ...
    # 服务的请求url（ip:port），方便本地调试
    url: ...
```

## 服务端配置

写api的时候建议这样写，方便以后调用者调试，调用方通过Configuration注入：

```java
@FeignClient(value = "${server.pay.serviceId:oasis-ct-pay-server}", url = "${server.pay.url:}")
@RequestMapping("/pay-server")
public interface PayCenterInterface {
}
```

接口是否配置 @FeignClient，调用方也有不同的配置方式：

```java

@Configuration
@EnableDiscoveryClient
// 1) 支付中心配置了 @FiegnClient，会自动扫描指定包下的 接口
@EnableFeignClients({"cn.oasis.pay.service"})
public class ServerConfig {

    // 2) PayCenterInterface 没有配置 @FeignClient
    @FeignClient(value = "${server.pay.serviceId:oasis-ct-pay-server}", url = "${server.pay.url:}")
    public interface PayServer extends PayCenterInterface {

    }

}

```

## Question

- 偶尔会发现接口重复定义的错误(@FeignClient)

    微服务接口(@FeignClient)定义的时候, 不要在类名上加 `@RequestMapping`, spring会扫描该注解, 并将所有方法映射 `mappingRegistry` 里
    
    原理:
    
    `org.springframework.web.servlet.handler.AbstractHandlerMethodMapping#initHandlerMethods`
    获取所有的bean, 筛选出controller方法, 见`org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping#isHandler`,
    所有 `@Controller` **或** `@RequestMapping` 的bean都将会扫描到, 这里解答了上面接口重复定义的问题,
    注册mapping: `org.springframework.web.servlet.handler.AbstractHandlerMethodMapping#registerHandlerMethod`
    