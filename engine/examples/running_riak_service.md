---
description: Build a Docker image with Riak pre-installed
keywords: docker, example, package installation, networking,  riak
title: Dockerize a Riak service
---

The goal of this example is to show you how to build a Docker image with
Riak pre-installed.

## 创建一个 Dockerfile

首先，创建一个空的文件，命名为 `Dockerfile`:

    $ touch Dockerfile

然后，定义一个构建镜像的基础镜像。我们会使用可以直接从[Docker Hub](https://hub.docker.com)上面获取的[Ubuntu镜像](https://hub.docker.com/_/ubuntu/) (tag:`trusty`)。

    # Riak
    #
    # VERSION       0.1.1
    # 使用由 dotCloud 提供的 Ubuntu 基础镜像
    FROM ubuntu:trusty
    MAINTAINER Hector Castro hector@basho.com

之后，我们会安装 curl 工具，该工具用来下载镜像仓库的初始化脚本，在下载脚本之后运行该脚本。

    # 在执行 apt-get update 命令之前安装 Riak 仓库，所以apt-get 的更新可以在一个步骤中就能完成
    RUN apt-get install -q -y curl && \
        curl -fsSL https://packagecloud.io/install/repositories/basho/riak/script.deb | sudo bash

然后我们安装并且初始化一些依赖服务：

 - `supervisor` 用来管理 Riak 的进程
 - `riak=2.0.5-1` 指明 Riak 的包版本为 2.0.5

<!-- -->

    # 然后我们安装并且初始化一些依赖服务
    RUN apt-get update && \
        apt-get install -y supervisor riak=2.0.5-1

    RUN mkdir -p /var/log/supervisor

    RUN locale-gen en_US en_US.UTF-8

    COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf

在此之后，我们会修改 Riak 的配置项:

    # 配置 Riak 接受任意IP地址的连接
    RUN sed -i "s|listener.http.internal = 127.0.0.1:8098|listener.http.internal = 0.0.0.0:8098|" /etc/riak/riak.conf
    RUN sed -i "s|listener.protobuf.internal = 127.0.0.1:8087|listener.protobuf.internal = 0.0.0.0:8087|" /etc/riak/riak.conf

然后我们对外暴露 Riak 的 Protocol Buffers 协议 和 HTTP接口：

    # 外暴露 Riak Protocol Buffers 和 HTTP 接口
    EXPOSE 8087 8098

最后，使用 `supervisord` 启动 Riak：

    CMD ["/usr/bin/supervisord"]

## 创建一个 supervisord 的配置文件

创建一个空的 `supervisord.conf` 文件. 确保该文件和您的 `Dockerfile` 处于同一个文件夹层级中。

    touch supervisord.conf

在该文件中，填写以下程序配置定义:

    [supervisord]
    nodaemon=true

    [program:riak]
    command=bash -c "/usr/sbin/riak console"
    numprocs=1
    autostart=true
    autorestart=true
    user=riak
    environment=HOME="/var/lib/riak"
    stdout_logfile=/var/log/supervisor/%(program_name)s.log
    stderr_logfile=/var/log/supervisor/%(program_name)s.log

## 创建 Riak 的 Docker 镜像

现在您能够创建一个 Riak的 Docker 镜像了：

    $ docker build -t "<yourname>/riak" .

## 接下来的步骤

Riak 是一个分布式数据库。许多线上部署方案占用 [至少5个节点](
http://basho.com/why-your-riak-cluster-should-have-at-least-five-nodes/)。
请查看 [docker-riak](https://github.com/hectcastro/docker-riak) 项目的详情来了解如何通过 Docker 和 Pipework 来部署一个 Riak 集群。
