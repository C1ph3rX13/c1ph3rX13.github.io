---
title: Docker CP Files
date: 2023-09-14T16:55:51+08:00
draft: false
url: /posts/2023-09-14/Docker-CP-Files
tags:
  - Docker
  - Docker-CP-Files
slug: English-Preview
---
> Docker CP Files
<!--more-->

# 概述

当发生容器安全事件时，需要从容器或镜像中提取恶意文件进行分析和处理

# docker cp

从镜像运行启动一个容器，然后使用docker cp命令从容器中提取文件到宿主机

```bash
# Usage
docker cp <容器名称或ID>:<容器内文件路径> <主机目标路径>

# 运行容器
docker run -d --name test test:v1.0
# 提取文件
docker cp test:/tmp/evil.sh  /tmp/eill.sh
# 删除容器
docker rm test
```

# docker export

`docker export` 命令可以将整个容器转储成一个 tar 归档文件，包括容器的文件系统和元数据信息

使用 `tar` 工具解压缩该文件，并提取所需文件

```bash
docker export <容器名称或ID> > <目标文件名>.tar
```

`docker save` 命令用于将 Docker 镜像保存为 tar 归档文件，会将镜像及其所有依赖的镜像层保存到单个文件中

```bash
# Usage
docker save <镜像名称> -o <输出文件名>.tar

docker save -o test.tar test:v1.0
tar -xvf test.tar 
tar -xvf cdbef1ee1b9602e5bd6c1897f0eb4f32c64380e97e0d456e85f7c0920b4d9e7b/layer.tar
eill.sh
```

# docker inspect

`docker inspect` 命令用于获取 Docker 容器、镜像或网络等对象的详细信息

通过执行 `docker inspect` 命令，可以查看与特定 Docker 对象相关联的所有元数据和配置

```bash
# Usage
docker inspect <镜像名称>
# 获取镜像的特定信息
docker inspect --format={{.GraphDriver}} <镜像名称>
docker inspect -f {{.GraphDriver}} <镜像名称>

docker inspect --format={{.GraphDriver}} test:v1.0
# Output
{map[LowerDir:/var/lib/docker/overlay2/3312fb65849f665a77d575b902532a7036e0f6184dcf346b4331fa1470a9ee8a/diff:/var/lib/docker/overlay2/e62491a257046f00defcaf4a27a8a5a947d7bcd812fe5ac78fa574233d8fcd32/diff:/var/lib/docker/overlay2/eb534d91f9fcd7dc072a72993c70fbdcc0bdde2599264affe47c534715844ae0/diff MergedDir:/var/lib/docker/overlay2/9ee7352980d88818e2246c5c5dc591732ec71c145608d884e094688b1d87b0c9/merged UpperDir:/var/lib/docker/overlay2/9ee7352980d88818e2246c5c5dc591732ec71c145608d884e094688b1d87b0c9/diff WorkDir:/var/lib/docker/overlay2/9ee7352980d88818e2246c5c5dc591732ec71c145608d884e094688b1d87b0c9/work] overlay2}
# 关注 UpperDir path
UpperDir:path

cd UpperDir:path
```

