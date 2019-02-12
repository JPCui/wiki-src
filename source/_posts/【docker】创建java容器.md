---
title: 【docker】创建java容器
date: 2018-11-05 15:11:52
tags:
  - docker
---
## 创建jre镜像

- Dockerfile / jre8

```
FROM daocloud.io/library/java:8u111-jre-alpine
MAINTAINER 624498030@qq.com
ARG DOCKER_HOST_IP
ENV DOCKER_HOST_IP=$DOCKER_HOST_IP
RUN echo "docker host ip found1: $DOCKER_HOST_IP"
```

- 构建 oasis/java 的镜像

> 因为consul服务里要用到服务本身的内网IP（区别于镜像内部IP），
> 所以单独构建了 oasis/java 的镜像，将服务器IP映射到容器内

把本机外网IP映射到镜像内

`docker build --tag=oasis/java:latest --build-arg DOCKER_HOST_IP=$(ifconfig eth0 | grep inet | awk '{print $2}') .`

## 构建容器

自行替换

- ${service-name} 服务名称
- ${outer-port}:${inner-port} 服务器端口:容器内端口
- ${container-name} 容器名字
- ${hotname} 主机名
- ${project-name} 项目打包后的名字

```
version: '2'
services:
  ${service-name}:
    image: oasis/java:latest
    ports:
      - "${outer-port}:${inner-port}"
      - "${debug-port}:${debug-port}"
    container_name : ${container-name}
    hostname: ${hostname}
    mem_limit: 3000m
    stdin_open: true
    tty: true
    working_dir: /opt/app
    command: java -Dfile.encoding=UTF-8 -Duser.timezone=GMT+08 -Dspring.profiles.active=prod -jar -Xmx512M ${project-name}.jar
    volumes:
      - /data1/auto_upline/${project-name}:/opt/app
      - /etc/localtime:/etc/localtime:ro
```

