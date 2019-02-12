---
title: 【consul】docker配置
date: 2018-11-05 15:11:52
tags:
  - consul
---

# server

> docker run -d --name=consul_service --net=host -e 'CONSUL_LOCAL_CONFIG={"skip_leave_on_interrupt": true}' consul agent -server -bind={server.ip} -join={master.ip} -ui -client 0.0.0.0

# client

> docker run -d --name=consul_client --net=host -e 'CONSUL_LOCAL_CONFIG={"skip_leave_on_interrupt": true}' consul agent -bind={server.ip} -join={master.ip} -ui -client 0.0.0.0

