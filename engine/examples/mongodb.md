---
description: Creating a Docker image with MongoDB pre-installed using a Dockerfile and sharing the image on Docker Hub
keywords: docker, dockerize, dockerizing, article, example, docker.io, platform, package, installation, networking, mongodb, containers, images, image, sharing, dockerfile, build, auto-building,  framework
title: Dockerize MongoDB
---

## Introduction

在本例中，我们将学习如何利用预先安装的MongoDB构建Docker镜像，以及如何将镜像push到[Docker Hub registry](https://hub.docker.com) 上分享给其他人。

> **Note:** 本篇概览会介绍构建一个MongoDB容器的方法，但是你应该更使用用[Docker Hub]( https://hub.docker.com/_/mongo/)
上的官方镜像

使用Docker和容器部署[MongoDB](https://www.mongodb.org/) 实例有一些好处，例如：

 - 对MongoDB实例更易维护，配置性更高 
 - 在毫秒级别内即可准备运行和启动
 - 基于全局共享和可访问的镜像

> **Note:** 如果你 **_不_** 喜欢`sudo`,你可以看[*Giving non-root access*](../installation/binaries.md#giving-non-root-access)

## 创建MongoDB的Dockerfile

开始创建我们的 `Dockerfile` 然后构建它：

```bash
$ nano Dockerfile
```

虽是可选，我们可以很方便地在`Dockerfile`的开头有一些关于这个镜像用途的解释：

```dockerfile
    # 将MongoDB容器化：用于构建MongoDB镜像的Dockerfile
    # 基于 ubuntu:latest, 安装MongbDB依照如下说明：
    # http://docs.mongodb.org/manual/tutorial/install-mongodb-on-ubuntu/
```


> **Tip:** `Dockerfile`非常灵活，但是编写它们必须遵循一个明确的格式。第一条必须声明一个你的MongoDB镜像的*父镜像* 的镜像名。

我们会用[Docker Hub Ubuntu](https://hub.docker.com/_/ubuntu/)仓库提供的最新版的Ubuntu，构建我们的镜像

```dockerfile
# Format: FROM    repository[:version]
FROM       ubuntu:latest
```

接下来，我们会声明`Dockerfile`的维护者(MAINTAINER)：

```dockerfile
# Format: MAINTAINER Name <email@addr.ess>
MAINTAINER M.Y. Name <myname@addr.ess>
```

> **Note:** 虽然Ubuntu系统有MongoDB包，但是他们很有可能是非常老的版本，因此在本例中，我们使用官方的MongoDB包。

```dockerfile
# Installation:
# Import MongoDB public GPG key AND create a MongoDB list file
RUN apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 7F0CEB10
RUN echo "deb http://repo.mongodb.org/apt/ubuntu "$(lsb_release -sc)"/mongodb-org/3.0 multiverse" | tee /etc/apt/sources.list.d/mongodb-org-3.0.list
```

在初始化准备后，我们可以更新我们的包，再安装MongoDB。

```dockerfile
# Update apt-get sources AND install MongoDB
RUN apt-get update && apt-get install -y mongodb-org
```

> **提示:** 如果你要安装特定版本的MongoDB，你可以将需要安装的包附带上版本号。例如：
> 
```dockerfile
RUN apt-get update && apt-get install -y mongodb-org=3.0.1 mongodb-org-server=3.0.1 mongodb-org-shell=3.0.1 mongodb-org-mongos=3.0.1 mongodb-org-tools=3.0.1
```

MongoDB需要一个存放数据的目录，我们把创建它作为我们安装指令的最后一步。

```dockerfile
# Create the MongoDB data directory
RUN mkdir -p /data/db
```

最后，我们设置`ENTRYPOINT`，它会告诉Docker 在通过我们的MongoDB镜像启动的容器中运行`mongod`。关于端口，我们会用`EXPOSE`指令。

```dockerfile
# Expose port 27017 from the container to the host
EXPOSE 27017
# Set usr/bin/mongod as the dockerized entry-point application
ENTRYPOINT ["/usr/bin/mongod"]
```

保存文件，我们来构建镜像。

> **Note:**  所有版本的`Dockerfile`可以在 [这里](https://github.com/docker/docker.github.io/blob/master/engine/examples/mongodb/Dockerfile)查看。

## 构建MongbDB的Docker镜像

有了我们的`Dockerfile`， 我们就可以用Docker构建MongoDB镜像。如果您不是用于测试，我们推荐给 `docker build` 命令传 `--tag` 参数，用于给镜像打标签的。

```dockerfile
# Format: docker build --tag/-t <user-name>/<repository> .
# Example:
$ docker build --tag my/repo .
```
一旦这个命令下发，Docker会通过 `Dockerfile` 构建镜像. 最终镜像会被打上标签 `my/repo`.

## 将MongbDB镜像push到Docker Hub

所有的Docker镜像仓库都可以通过 `docker push` 命令被推送和分享到 [Docker Hub](https://hub.docker.com) . 不过前提是您需要登录。

```bash
# Log-in
$ docker login

Username:
..
```

```bash
# Push the image
# Format: docker push <user-name>/<repository>
$ docker push my/repo

The push refers to a repository [my/repo] (len: 1)
Sending image list
Pushing repository my/repo (1 tags)
..
```

## 使用MongoDB镜像

使用我们创建的MongoDB镜像，我们可以启动一个甚至多个MongoDB实例作为守护进程。

```bash
# Basic way
# Usage: docker run --name <name for container> -d <user-name>/<repository>
$ docker run -p 27017:27017 --name mongo_instance_001 -d my/repo

# Dockerized MongoDB, lean and mean!
# Usage: docker run --name <name for container> -d <user-name>/<repository> --noprealloc --smallfiles
$ docker run -p 27017:27017 --name mongo_instance_001 -d my/repo --smallfiles

# Checking out the logs of a MongoDB container
# Usage: docker logs <name for container>
$ docker logs mongo_instance_001

# Playing with MongoDB
# Usage: mongo --port <port you get from `docker ps`>
$ mongo --port 27017

# If using docker-machine
# Usage: mongo --port <port you get from `docker ps`>  --host <ip address from `docker-machine ip VM_NAME`>
$ mongo --port 27017 --host 192.168.59.103
```

> **Tip:** 如果你想要在同一个Docker Engine上运行两个容器，你需要将暴露的端口映射到宿主机上的两个不同端口。

```bash
# Start two containers and map the ports
$ docker run -p 28001:27017 --name mongo_instance_001 -d my/repo

$ docker run -p 28002:27017 --name mongo_instance_002 -d my/repo

# 现在你可以用这两个端口连接任意一个MongoDB实例
$ mongo --port 28001

$ mongo --port 28002
```

 - [Linking containers](../userguide/networking/default_network/dockerlinks.md)
 - [Cross-host linking containers](../admin/ambassador_pattern_linking.md)
 - [Creating an Automated Build](/docker-hub/builds/)