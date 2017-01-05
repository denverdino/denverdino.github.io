---
alias:
- /reference/api/hub_registry_spec/
- /userguide/image_management/
description: Documentation for docker Registry and Registry API
keywords: docker, registry, api,  hub
title: Image management
---

Docker Engine提供了一个客户端，可以使用这个客户端通过命令行或者构建的过程来创建镜像。可以在容器中执行这些镜像或者发布镜像给其他人使用。保存你创建的镜像，搜索你想要的镜像，发布其他人可能会使用的镜像是镜像管理的内容。

这一部分是一个概览，介绍了Docker针对镜像管理提供的主要特性和产品

## Docker Hub

Docker Hub 是一个集中管理用户账户，镜像，公共名空间的平台。它包含以下组件：
 - Web UI界面
 - 元信息存储（评论，星标，公共仓库列表）
 - 鉴权服务
 - 令牌化

Docker Hub 只有一个部署，由Docker Inc.公司运营和管理。公共的Docker Hub能满足个人开发者和小公司的需求。

## Docker Registry 和 Docker Trusted Registry

Docker Registry 是Docker生态中的一个组件。registry是一个存储和内容分发系统，包含了带有名称的Docker镜像，并且有多个不同的tag。
例如，镜像`distribution/registry`可以有两个tag，`2.0` 和 `latest`。用户与registry进行交互的时候，使用docker push和docker pull等命令，例如
`docker pull myregistry.com/stevvooe/batman:voice`。

Docker Hub有它自己的registry，同样是由Docker运营和管理的。有其他的获得registry的途径，你可以购买[Docker Trusted Registry](/docker-trusted-registry)产品，将该产品部署在你公司的网络上。
另外，你可以使用Docker Registry组件来搭建你自己私人的registry，关于使用registry的信息，请查看[Docker Registry](/registry)概览。


## 内容可信

当在网络系统中传输数据时，*信任*是一个需要特别考虑的问题。特别是当我们在例如互联网这样的不受信环境中通信时，
保证一个系统中所有数据的完整性和发布者的可信是很关键的。当你使用Docker从registry拉取或者推送镜像时，
内容可信将给你通过任意渠道验证数据完整性和数据发布者可信的能力。


目前只有公共Docker Hub提供[内容可信](../../security/trust/index.md)的能力。Docker Trusted Registry 或者
私有的registry还不具有这样的能力。
