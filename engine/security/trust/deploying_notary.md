---
description: Deploying Notary
keywords: trust, security, notary, deployment
title: Deploying Notary Server with Compose
---

最简单的部署Notary Server方式是通过Docker Compose。遵循本页的步骤，你需要先 [安装 Docker Compose](/compose/install.md).

1. 克隆  Notary 仓库

        git clone git@github.com:docker/notary.git

2. 用示例证书构建和启动Notary Server.

        docker-compose up -d


    了解更多关于部署Notary Server请参见 [instructions to run a Notary service](/notary/running_a_service.md) 或者 https://github.com/docker/notary
3. 在你尝试和Notary server交互前，确保你的Docker或 Notary 客户端信任 Notary Server的证书。

根据你的需要，查看关于 [Docker](../../reference/commandline/cli.md#notary) 或者
 [Notary](https://github.com/docker/notary#using-notary)的说明 。

## 如果你希望在生产环境使用Notary

请在Notary Server有一个官方稳定版后回来检查操作说明。想抢先看如何在生产环境部署Notary，可以查看https://github.com/docker/notary。