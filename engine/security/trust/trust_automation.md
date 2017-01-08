---
description: Automating content push pulls with trust
keywords: trust, security, docker,  documentation, automation
title: Automation with content trust
---

你的拉取镜像、构建镜像的自动化系统也可以和信任一起工作。任何自动化环境需要在处理镜像前设置`DOCKER_TRUST_ENABLED`，要么手动要么通过脚本的方式。

## 绕过口令的输入

让工具封装Docker并推送信任内容，有两个环境变量让你提供口令而不需要expect脚本或输入他们：

 - `DOCKER_CONTENT_TRUST_ROOT_PASSPHRASE`
 - `DOCKER_CONTENT_TRUST_REPOSITORY_PASSPHRASE`

Docker尝试使用环境变量的内容作为秘钥的口令。比如，一个镜像发布者可以导出仓库`target` and `snapshot`口令：

```bash
$  export DOCKER_CONTENT_TRUST_ROOT_PASSPHRASE="u7pEQcGoebUHm6LHe6"
$  export DOCKER_CONTENT_TRUST_REPOSITORY_PASSPHRASE="l7pEQcTKJjUHm6Lpe4"
```

然后，当推送一个新的tag，Docker客户端不会请求这些口令而会自动签名：

```bash
$  docker push docker/trusttest:latest
The push refers to a repository [docker.io/docker/trusttest] (len: 1)
a9539b34a6ab: Image already exists
b3dbab3810fc: Image already exists
latest: digest: sha256:d149ab53f871 size: 3355
Signing and pushing trust metadata
```

当直接通过Notary客户端工作，它会使用[own set of environment variables](/notary/reference/client-config.md#environment-variables-optional).

## 通过内容信任构建

你也可以使用信任安全构建。在运行`docker build`命令之前，你需要手动或通过脚本的方式设置环境变量`DOCKER_CONTENT_TRUST`。考虑下面的简单Dockerfile:

```Dockerfile
FROM docker/trusttest:latest
RUN echo
```

`FROM` tag 在拉取一个签名镜像. 你不能构建这个镜像除非`FROM` 在本地或者被签名。考虑到tag `latest`的内容信任数据存在，下列的构建可以成功:
```bash
$  docker build -t docker/trusttest:testing .
Using default tag: latest
latest: Pulling from docker/trusttest

b3dbab3810fc: Pull complete
a9539b34a6ab: Pull complete
Digest: sha256:d149ab53f871
```

如果内容信任开启，从Dockerfile的构建依赖一个没有信任内容的tag，会导致构建命令失败：

```bash
$  docker build -t docker/trusttest:testing .
unable to process Dockerfile: No trust data for notrust
```

## 相关信息

* [Docker 内容信任](content_trust.md)
* [管理内容信任秘钥](trust_key_mng.md)
* [内容信任-代理](trust_delegation.md)
* [在内容信任沙箱中运行](trust_sandbox.md)