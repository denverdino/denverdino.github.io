---
description: Installation instructions for Docker on cloud.
keywords: cloud, docker, machine, documentation,  installation
redirect_from:
- /engine/installation/cloud/cloud/
title: Choose an installation method
---

你可以在使用Docker支持的操作系统的云平台上安装Docker Engine。这些系统可以是Linux的某个版本，Mac 或 Windows。

你可以使用两种方式进行安装：

* 在云端手动安装（创建云主机，然后安装Docker Engine）
* 使用Docker Machine创建云主机

## 在云主机中手动安装Docker Engine

安装步骤如下:

1. 创建云服务账号，查阅创建主机的相关文档，了解创建过程。

2. 选择云主机的操作系统

3. 了解在该系统中安装Docker的要求和步骤. 可以在[安装Docker Engine](../index.md) 中查询支持的系统和安装指南.

4. 创建Docker支持操作系统的云主机，然后按着说明一步步在上面安装Docker.

[示例 (AWS): 手动在云服务商环境中安装](cloud-ex-aws.md) 中介绍了如何创建 <a href="https://aws.amazon.com/" target="_blank"> 亚马逊云服务 (AWS)</a> EC2 实例, 并在上面安装Docker Engine.


## 使用Docker Machine创建云主机

Docker Machine插件提供了对多个流行的云平台的支持。你可以使用Machine在这些平台创建一台或多台安装好Docker环境的主机。
使用Docker Machine，你可以用相同的接口创建云主机并在上面部署Docker Engine
你可以使用 `docker-machine create` 命令创建，driver参数指定云服务商，此外还需要添加账号验证，安全验证及更多配置信息

[示例: 使用 Docker Machine 创建云主机](cloud-ex-machine-ocean.md) 提供一步步指导来安装 Docker Machine 及 创建安装Docker环境的 <a href="https://www.digitalocean.com/" target="_blank">Digital Ocean</a>主机.

## 继续阅读
* [示例: 手动在云服务商环境中安装](cloud-ex-aws.md) (AWS EC2)

* [示例: 使用Docker Machine创建云主机](cloud-ex-machine-ocean.md) (Digital Ocean)

* 平台支持情况请查阅 [安装 Docker Engine](../index.md).

* 安装Docker完成后, 了解[Docker 用户指南](../../userguide/intro.md).