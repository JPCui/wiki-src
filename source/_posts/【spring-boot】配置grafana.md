---
title: 【spring-boot】配置grafana
date: 2018-11-05 15:11:52
tags:
  - spring-boot
---

## 项目配置(spring-boot:1.5.x)

```xml
<dependecies>
    <!-- 1.1.1 -->
    <!-- prometheus相关依赖 -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-spring-legacy</artifactId>
        <version>RELEASE</version>
    </dependency>
    
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-registry-prometheus</artifactId>
        <version>RELEASE</version>
    </dependency>

    <!-- Actuator (with security enabled) -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
</dependecies>
```

## grafana

![](../assets/grafana-01.png)

![](../assets/grafana-02.png)

![](../assets/grafana-03.png)

Dashboard for Spring Boot 1.x applications, using Micrometer and Prometheus.

### Features
- Overall status
- API stats
- Tomcat
- JVM

### 变量
Only one variable is declared in Grafana:
在该dashboard中有一个变量 `$job`, 用来标识job名, 等同于`prometheus`中的`job_name`, 
不过在监控`consul`的服务时, 用来标识服务名, 即项目里配置的 `spring.application.name`

配置方式:

{dashboard} -> settings -> Variables

## promethues 配置

```yaml
# 监控consul
scrape_configs:
  # cousul_services
  - job_name: 'consul_services'
    consul_sd_configs:
      - server: '${consul-server}:8500'
        services: ['${server.name}']
  
    # 修改抓取route
    relabel_configs:
        - source_labels: ['__metrics_path__']
          regex:         '/metrics'
          target_label:  __metrics_path__
          replacement:   '/prometheus'
  
  # 通过consul监控所有服务
  - job_name: 'consul_server'
    metrics_path: '/prometheus'
    consul_sd_configs:
      - server: '${consul-server}:8500'
        services: ['${server.name}']
    # 修改抓取route
    relabel_configs:
        - source_labels: ['__meta_consul_service']
          regex:         '(.*)'
          replacement:   '$1'
          action:        replace
          target_label:  'job'

  # 监控普通spring-boot项目
  # for https
  - job_name: my-app-prod
    scheme: https
    basic_auth:
      username: your_actuator_user
      password: your_actuator_password
    metrics_path: /MyApp/actuator/prometheus
    static_configs:
      - targets:
          - "your_hostname:your_port"
  
  # for http
  - job_name: my-app-prod
    scheme: http
    metrics_path: /MyApp/actuator/prometheus
    static_configs:
      - targets:
          - "your_hostname:your_port"
```

## 效果预览

![](../assets/grafana-preview.png)
