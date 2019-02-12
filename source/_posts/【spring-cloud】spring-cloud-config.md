---
title: 【spring-cloud】spring-cloud-config
date: 2018-11-05 15:11:52
tags:
  - spring-cloud
---
## server

- 依赖
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>
```

- yml配置
```yaml
spring:
  cloud:
    config:
      server:
        git:
          # 配置的git路径
          uri: http://xxx/oasis-ct/oasis-ct-config.git
          # 查找路径
          # 默认查找根目录, 而且只查找一级目录
          search-paths: repo
      profile: ${spring.profiles}

```

> 如果想把配置中心以服务的形式配置到consul中, 可以按照服务注册的形式进行配置

## client

- 依赖
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
```

- yml配置
```yaml
spring:
    # 配置中心
    config:
      # 1. 直接访问方式
      uri: `{config-server}`
      # 2. 服务注册到consul, 需要按下面的方式配置      
#      discovery:
#        enabled: truegetDeliveryInfo
#        service-id: oasis-ct-config

      # 由于咱们的项目名比较长, 这里可以重命名, 
      # 同时config服务里的配置文件的文件名命名方式: medicine-{profile}.yml
      name: medicine
      profile: ${spring.profiles}
      fail-fast: true
```