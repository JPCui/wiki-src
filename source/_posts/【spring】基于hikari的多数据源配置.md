---
title: 【spring】基于hikari的多数据源配置
date: 2018-11-05 15:11:52
tags:
  - hikari
  - spring
---


> 问题：`mgmt`项目目前配置了多数据源，当手动配置 `HikariDataSource` 时，会出现 DataSource 使用的是 Tomcat 的 `DataSource`

# 使用 Hikari 实现多数据源

## 检查数据源是否为 Hikari

`ApplicationContext.getBeansOfType(DataSource.class)` 是否为 `HikariDataSource`，并在debug时，查看属性`pool`是否为 `HikariPool-x` ，

如果检查不是 Hikari（有可能是 tomcat.DataSource），可按如下步骤配置

## 配置 HikariDataSource

- DataSourceConfig

```

    # 一个默认的 DataSource
    @ConfigurationProperties(prefix = "spring.datasource.hikari")
    public DataSource dataSource(DataSourceProperties properties) {
       return properties.initializeDataSourceBuilder().type(HikariDataSource.class).build();
    }

    # 第二个 DataSource
    @ConfigurationProperties(prefix = "spring.datasource2.hikari")
    public DataSource dataSource(DataSourceProperties properties) {
       return properties.initializeDataSourceBuilder().type(HikariDataSource.class).build();
    }

```

- application.yml

```yml

spring:

  # 这里使用了模板，可以省去一些公共配置
  tpl_datasource: &tpl_datasource
    type: com.zaxxer.hikari.HikariDataSource
    driver-class-name: com.mysql.jdbc.Driver
    url: $url
    username: $username
    password: $pwd
    maximum-pool-size: 23
    connection-timeout: 55000
    idle-timeout: 33000
    testOnBorrow: true
    maxLifetime: 11000                  #30分钟，服务器wait_timeout设置为30分钟了

  # 默认的datasource，其他ds会继承该配置，有点像模板
  datasource:
    <<: *tpl_datasource
    username: root                      # 其他ds会继承
    password: oasisadmin                # 其他ds会继承
    hikari:
      jdbc-url: jdbc:mysql://xxx/oasis_mgmt?useUnicode=true&characterEncoding=utf8&useSSL=false&autoReconnect=true&autoReconnectForPools=true
      connection-init-sql: select 1
      leakDetectionThreshold: 5000      #2s 后connection还不回归pool，则提示可能的connection leak
      connectionTimeout: 5000 #5s,get connection from pool
      maxLifetime: 21000                #30分钟，服务器wait_timeout设置为30分钟了
      maximum-pool-size: 24
      data-source-properties:
        cachePrepStmts: true
        prepStmtCacheSize: 250
        prepStmtCacheSqlLimit: 2048
        useServerPrepStmts: true

  datasource_mom:
    <<: *tpl_datasource
    hikari:
      jdbc-url: jdbc:mysql://xxx/oasis_mom_formal?useUnicode=true&characterEncoding=utf8&useSSL=false&autoReconnect=true&autoReconnectForPools=true
      connection-init-sql: select 1
      leakDetectionThreshold: 5000      #2s 后connection还不回归pool，则提示可能的connection leak
      connectionTimeout: 5000           #5s,get connection from pool
      maxLifetime: 21000                #30分钟，服务器wait_timeout设置为30分钟了
      maximum-pool-size: 24
      data-source-properties:
        cachePrepStmts: true
        prepStmtCacheSize: 250
        prepStmtCacheSqlLimit: 2048
        useServerPrepStmts: true

```

- 由于 hikari使用的是 jdbc-url，所以 `spring.datasource.hikari.jdbc-url` 在每个 ds 中都要单独配置

- 由于上一条，所以 hikari 的配置无法使用 tpl模板 的方式让其他地方使用，`spring.datasource.hikari.*` 也都要单独配置

- `spring.datasource.*` 里面的配置，所有ds默认都会使用，比如 `hikari.jdbc-url` 如果没有配置，则默认使用 `spring.datasource.url`

# reference

- org.springframework.boot.autoconfigure.jdbc.DataSourceConfiguration
- [Configure HikariCP with Spring Boot JPA Hibernate and PostgreSQL as a database](https://gist.github.com/rhamedy/b3cb936061cc03acdfe21358b86a5bc6)
