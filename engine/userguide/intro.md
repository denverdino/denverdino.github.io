---
description: Introduction to user guide
identifier: engine_guide_intro
keywords: docker, introduction, documentation, about, technology, docker.io, user, guide, user's, manual, platform, framework, home, intro
redirect_from:
- /userguide/
title: Engine user guide
---

本指南将带你了解下Docker Engine的基本知识和将Docker Engine部署到你的环境。你将学会如何使用
Docker去：
* Docker化你的应用。
* 运行你自己的容器。
* 构建Docker镜像。
* 跟其他人分享你的Docker镜像。
* 其他Docker相关知识

本指南分为几大部分，这几部分将带领你学习Docker Engine的基本知识和其他支持Docker Engine的产品。

## Docker化应用程序："Hello world"

*我如何在容器中运行应用程序呢？*

Docker Engine提供了容器化的平台来赋能你的应用。学习如何Docker化你的应用和运行它们：

访问[容器化应用程序](../tutorials/dockerizing.md)


## 玩转容器

*如何管理我的容器*


一旦你开始将你的应用程序运行于Docker 容器之上，你将学会如何管理那些容器。要掌握如何
查看，监控以及管理容器：

访问[Working with Containers](../tutorials/usingdocker.md)。

## 玩转Docker 镜像

*我如何能够访问，分享，构建我的镜像？*

一旦你学会如何使用Docker，该是进入下一步，学习如何使用Docker打造属于自己的应用镜像的时候了。

访问 [玩转镜像](../tutorials/dockerimages.md)

## 容器网络互联

直到目前为止，我们已经了解了如何在Docker容器中构建单个的应用程序。现在我们开始学习如何使用Docker网络
来构建整个应用栈

访问 [容器网络互联](../tutorials/networkingcontainers.md)。

## 管理容器中的数据

现在我们知道了如何让Docker容器通信，接下来我们学习如何在容器中管理数据，数据卷和挂载点

访问[管理容器中的数据](../tutorials/dockervolumes.md) 

## 管理Docker对象的元数据(标签)

标签是用来将元数据应用到Docker 对象中的一种机制，这样的Docker对象包括：
- 镜像
- 容器
- 本地Docker守护进程
- 数据卷
- 网络对象
- Swarm 节点
- Swarm 服务

你可以使用标签来组织你的镜像，记录你的许可信息，注解容器，数据卷，网络对象之间的关系，或者其他对你的应用或业务有价值的信息。

访问 [管理Docker对象标签](labels-custom-metadata.md)。

## 补充Engine的Docker产品

通常来说，一项强大的技术会衍生出许多其他的发明，这样的发明会使得这项技术更加容易获取，使用，更加强大。这些衍生的发明都有一个共同的特征：它增强了核心的技术。
下面这些Docker的产品扩展了Docker Engine的功能。

### Docker Hub

Docker Hub是Docker的中心枢纽。它提供了公共的Docker镜像和提供服务来帮助你构建和管理你的Docker环境。了解更多：

访问 [使用Docker Hub](/docker-hub/)

### Docker Machine

Docker Machine 帮助你快速启动和运行Docker Engine。Docker Machine能在你的电脑，云提供商环境或者你的数据中心启动你的主机并且配置好Docker Engine，
然后配置好你的Docker客户端，并且以安全的方式与Docker Engine通信。

访问 [Docker Machine用户指南](/machine/)
Go to [Docker Machine user guide](/machine/).

### Docker Compose

Docker Compose 允许你定义应用的组件 -- 它们的容器，配置，link，数据卷 -- 全部在一个文件中。然后
一个命令就能做好各项准备并且启动你的应用程序。

访问 [Docker Compose用户指南](/compose/)。


### Docker Swarm

Docker Swarm 组合了多个Docker Engine并将其以一个虚拟Docker Engine的形式暴露出来。它像Docker Engine一样
支持标准的Docker API。因此任何需要配合Docker使用的工具可以透明的扩展到多个主机。

访问 [Docker Swarm用户指南](/swarm/)。

## 获得帮助

* [Docker 官方首页](https://www.docker.com/)
* [Docker Hub](https://hub.docker.com)
* [Docker 博客](https://blog.docker.com/)
* [Docker 文档](/)
* [Docker 入门指南](../getstarted/index.md)
* [Docker GitHub项目地址](https://github.com/docker/docker)
* [Docker 邮件列表](https://groups.google.com/forum/#!forum/docker-user)
* Docker IRC: irc.freenode.net and channel #docker
* [Docker Twitter账号](https://twitter.com/docker)
* 在 StackOverflow 上获得 [Docker相关 帮助](https://stackoverflow.com/search?q=docker)
