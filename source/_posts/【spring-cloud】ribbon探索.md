---
title: 【spring-cloud】ribbon探索
date: 2018-11-05 15:11:52
tags:
  - spring-cloud
---
## 配置

```
# 开启负载均衡重试
spring.cloud.loadbalancer.retry.enabled: true
ribbon:
  # 最多重试多少台服务
  MaxAutoRetriesNextServer: 2
  # 每台服务器最多重试次数，但是首次调用不包括在内
  MaxAutoRetries: 1
  # 连接超时时间
  ConnectTimeout: 1000
  # 读取超时时间
  ReadTimeout: 3000
```

如果针对某个服务做负载均衡, 可把 ribbon 配置放在该服务节点下, 比如: 

```
xxx-server:
  ribbon:
    # 最多重试多少台服务
    MaxAutoRetriesNextServer: 2
    # 每台服务器最多重试次数，但是首次调用不包括在内
    MaxAutoRetries: 1
    # 连接超时时间
    ConnectTimeout: 1000
    # 读取超时时间
    ReadTimeout: 3000
```

或者使用注解: @RibbonClients, @RibbonClient(name="xxx-server", configuration=XXXConfiguration.class)

## 配置类源码追溯

> 配置相关 key
> com.netflix.client.config.CommonClientConfigKey
>
> ribbon ClientConfig 构造器
> com.netflix.client.config.IClientConfig.Builder
>
> 默认值(除 connectTimeout, readTimeout 外, 其他配置都使用该类中的默认值, 前面两个属性在 RibbonClientConfiguration 已经声明)
> com.netflix.client.config.DefaultClientConfigImpl

> 💡:bulb:
> 这里需要注意一个地方
> connectTimeout, readTimeout 如果没有显式配置, 默认值为: 1000, 1000.
> > org.springframework.cloud.netflix.ribbon.RibbonClientConfiguration#DEFAULT_CONNECT_TIMEOUT = 1000
> > org.springframework.cloud.netflix.ribbon.RibbonClientConfiguration#DEFAULT_READ_TIMEOUT = 1000
> 初始化 RibbonClientConfig 的位置在 org.springframework.cloud.netflix.ribbon.RibbonClientConfiguration#ribbonClientConfig.
>
> 有配置的时候,自然使用配置的value, DefaultClientConfigImpl里的默认值然而并没有用到(ps: 应该在其他地方用到了: HttpClientRibbonConfiguration, OkHttpRibbonConfiguration)
>
> 这大概就是为什么我们服务(牵涉到三方调用)经常出现超时的原因

## 负载均衡规则

- **ClientConfigEnabledRoundRobinRule** 内嵌了一个 **RoundRobinRule**
  - **BestAvailableRule**         最优可用: 获取其中ActiveRequestsCount最小的服务
  - **PredicateBasedRule**        基于 Predicate 的规则 
    - **ZoneAvoidanceRule**       区域去除判定: 判断某个区域的运行性能是否可用, 提出不可用的区域(该区域下的所有服务)
    - **AvailabilityFilterRule**  可用性判定: 过滤掉连接数过多的server
- **RoundRobinRule**              轮询
  - **WeightedResponseTimeRule**  根据响应时间分配一个weight，响应时间越长，weight越小，被选中的可能性越低
  - **ResponseTimeWeightTimeRule** (??? use WeightedResponseTimeRule instead)
- **RandomRule** 随机
- **RetryRule**  重试规则, 失败之后会在指定时间内以subRule继续获取服务, 默认subRule=RoundRobinRule
  
规则配置方式(以 RetryRule 为例):

- properties方式

```
ribbon:
  NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RetryRule
```

- java config

```
@Bean
IRule rule() {
    RetryRule retryRule = new RetryRule();
    retryRule.setMaxRetryMillis(1000);
    retryRule.setRule(subRule());
    return retryRule;
}

IRule subRule() {
    return new RoundRobinRule();
}
```

默认规则: com.netflix.loadbalancer.AvailabilityFilteringRule

## 依赖说明

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
</dependency>

...

<dependencyManagement>
    <dependencies>
        <dependency>
          <groupId>org.springframework.cloud</groupId>
          <artifactId>spring-cloud-dependencies</artifactId>
          <version>Edgware.SR3</version>
          <type>pom</type>
          <scope>import</scope>
      </dependency>
    </dependencies>
</dependencyManagement>
```

## reference

- https://github.com/Netflix/ribbon/wiki/Getting-Started
