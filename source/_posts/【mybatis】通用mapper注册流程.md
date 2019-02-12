---
title: 【mybatis】通用mapper注册流程
date: 2018-11-05 15:11:52
tags:
  - mybatis
---

> https://github.com/abel533/Mapper
> /oasis-o2o/Mapper

## 通用mapper注册流程

使用咱们自己的mapper扩展插件时, 想自定义mapper实现批量插入(见:InsertListMapper), 发现无法实例化 `SpecialProvider`, 

`org.apache.ibatis.builder.BuilderException: Error invoking SqlProvider method`

研究其mapper注册流程发现, 在 `MapperScannerConfigurer` 中未配置 `mappers` 时, 默认注册 `tk.mybatis.mapper.common.Mapper`, 代码见 `tk.mybatis.mapper.mapperhelper.MapperHelper#ifEmptyRegisterDefaultInterface`, 如果指定了 `mappers`, 则在 `setProperties` 时就开始注册了, 代码见 `tk.mybatis.mapper.mapperhelper.MapperHelper#setProperties`