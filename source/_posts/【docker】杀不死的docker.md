---
title: 【docker】杀不死的docker
date: 2018-11-05 15:11:52
tags:
  - docker
---

Cannot restart container oasis-roomservice: Cannot kill container 9f34690345ddc450f8dcb60ed72fd0ceba3a9ce5196ba14e0a9b8e346082280a: connection error: desc = "transport: dial unix /var/run/docker/containerd/docker-containerd.sock: connect: connection refused": unknown

应该是内存不足导致的

- https://github.com/moby/moby/issues/36002
- https://github.com/jwilder/nginx-proxy/issues/1034
