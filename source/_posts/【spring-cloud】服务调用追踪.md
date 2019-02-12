---
title: 【spring-cloud】服务调用追踪
date: 2018-11-05 15:11:52
tags:
  - spring-cloud
---
服务监控——服务调用链追踪，帮助追踪每个请求的调用过程及每个过程的网络延迟、业务耗时等指标

服务调用追踪主要是根据 traceId, spanId, parentSpanId 在个服务日志上进行标记, 一个traceId标记一个请求, spanId标记链路, parentSpanId为上一条链路的spanId, 由traceId把所有的spanId串联起来, 在第一条链路上 traceId = spanId = parentSpanId.


## 配置步骤

### 配置 ELK （便于查询整个请求的处理日志）

  - 添加依赖

      ```
      <dependency>
          <groupId>org.springframework.cloud</groupId>
          <artifactId>spring-cloud-starter-sleuth</artifactId>
      </dependency>
      ```
  - 在 `logback-spring.xml` 的 `LOGSTASH` 节点下添加如下配置：

      ```
      <additionalField>
          <key>X-B3-ParentSpanId</key>
          <value>@{X-B3-ParentSpanId}</value>
      </additionalField>
      <additionalField>
          <key>X-B3-SpanId</key>
          <value>@{X-B3-SpanId}</value>
      </additionalField>
      <additionalField>
          <key>X-B3-TraceId</key>
          <value>@{X-B3-TraceId}</value>
      </additionalField>
      ```
  
      参考：[oasis-ct-product-server](/oasis-ct/oasis-ct-product/src/master/oasis-ct-product-server/src/main/resources/logback-spring.xml)

### 配置 zipkin

  - 添加依赖

      ```pom
      <dependency>
          <groupId>org.springframework.cloud</groupId>
          <artifactId>spring-cloud-sleuth-zipkin</artifactId>
      </dependency>
      ```

  - 在 `application-dev.yml` / `application-test.yml` 中配置：
  
      ```yml
      spring:
        zipkin:
          enabled: true
          # zipkin 部署在 op 服务器
          base-url: http://10.163.149.166:11001
        sleuth:
          enabled: true
          sampler:
            # 采样的比例
            percentage: 1.0
          scheduled:
            # 关闭 scheduled 方法的采样，否则 zipkin 上一堆异步任务的 trace，尤其是 consul watch task
            # Refer: [scheduled_annotated_methods]
            enabled: false
      ```
      
      如果某个环境没有配 zipkin, 会按照默认值进行请求zipkin, 所以最好配置enabled=false, 禁用zipkin:
      
      `spring.zipkin.enabled: false`
  
  - 检查配置是否成功
  
      访问任意一个接口，在 [zipkin] 中查看是否有该请求的记录

      查不到的原因：

        - 配置没有成功
        - 由于采样配置不是 1.0，导致没有采样(所谓采样, 就是当前日志会不会打到 zipkin 上)

  - 参考
  
      [oasis-ct-product-server](/oasis-ct/oasis-ct-product/src/master/oasis-ct-product-server/src/main/resources/config/application-dev.yml)

## 随机采样原理

参考: `org.springframework.cloud.sleuth.sampler.PercentageBasedSampler#isSampled`

- [generate-a-random-bitset-with-n-1s](http://stackoverflow.com/questions/12817946/generate-a-random-bitset-with-n-1s)

## 遗留问题

- 可以引入消息队列帮助解耦及提升效率

- 集成 ELK 之后，在打印非请求的日志里，`X-B3-*` 相关的字段显示的是 `X-B3-*_NOT_FOUND`
  
  理论上应该是 `-` 或 ` `（空），看源码 `com.cwbase.logback.JSONEventLayout#mdcSubst` ，是框架自己没有设置默认值的方法
  
  所以暂时先不处理，或以后改用 `FileAppender` + `logstash` 的方式收集日志
  
## 相关文档

- [spring-cloud-sleuth Reference](https://cloud.spring.io/spring-cloud-sleuth/1.3.x/single/spring-cloud-sleuth.html)

- spring-cloud-sleuth: [scheduled_annotated_methods](https://cloud.spring.io/spring-cloud-static/spring-cloud-sleuth/1.3.4.RELEASE/single/spring-cloud-sleuth.html#__scheduled_annotated_methods)

- [Spring Cloud Sleuth使用简介](https://www.jianshu.com/p/6d6b52c7624f)

- Spring Cloud Sleuth does not add trace/span related headers to the Http Response for security reasons. If you need the headers, see: [在http response 中添加 trace header](https://cloud.spring.io/spring-cloud-sleuth/1.3.x/single/spring-cloud-sleuth.html#_example)
