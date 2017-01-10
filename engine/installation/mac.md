---
description: Docker installation on macOS
keywords: Docker, Docker documentation, requirements, boot2docker, VirtualBox, SSH, Linux, osx, os x, macOS, Mac
title: Install Docker on macOS
---

在Mac上你可以使用两种方式安装Docker：

- [Docker for Mac](mac.md#docker-for-mac)
- [Docker Toolbox](mac.md#docker-toolbox)

## Docker for Mac

Docker for Mac是我们提供的最新的Mac安装方式。它以一个原生Mac应用方式运行，使用<a href="https://github.com/mist64/xhyve/" target="_blank">xhyve</a>来虚拟化Docker Engine环境和Docker进程对Linux 内核某型特性的要求。

参见[Getting Started with Docker for Mac](/docker-for-mac/)，了解下载、安装说明和所有Docker for Mac的相关内容。

**要求**

- Mac 必须是2010或之后推出的型号。要求Intel 硬件支持MMU虚拟化，如EPT

- macOS 10.10.3 Yosemite 或 更新

- 至少4GB内存

- 不能安装低于4.3.30的VirtualBox（同Docker for Mac不兼容）。如果安装过程中报错，请卸载旧版本的VirtualBox，并重试安装。

## Docker Toolbox

如果你的Mac不满足安装Docker for Mac的要求，请使用<a href="https://www.docker.com/products/docker-toolbox" target="_blank">get Docker Toolbox</a>

参见[Docker Toolbox Overview](/toolbox/overview.md) 中的说明来安装Docker Toolbox

Docker Toolbox并不在macOS本地运行Docker，它使用 `docker-machine` 来创建并连接虚拟机（VM），Docker跑在这台Linux虚拟机上，虚拟机运行在你的Mac上。

**要求**

在macOS 10.8 "Mountain Lion"或更新的系统中才能安装Docker Toolbox。完整的安装说明请见[Toolbox install instructions for Mac](/toolbox/toolbox_install_mac.md).


## 更多内容

* 如果你是Docker新手，[入门指南](../getstarted/index.md)中包含使用Docker命令、运行容器、构建镜像、使用Docker Hub等一系列内容。

* 你可以在[Docker Engine User Guide](../userguide/index.md)中的[Learn by example](../tutorials/index.md)部分查看到更多示例。

* 如果你对Kitematic GUI有兴趣，可以参考[Kitematic user guide](/kitematic/userguide/).

> **注意**: The Boot2Docker 前几个版本中就已经不再维护，推荐使用Docker Machine 或更新的Docker for Mac.