---
description: Docker installation on Microsoft Windows
keywords: Docker, Docker documentation, Windows, requirements, virtualbox,  boot2docker
title: Install Docker on Windows
---

在Windows上你可以使用两种方式安装Docker：

- [Docker for Windows](windows.md#docker-for-windows)
- [Docker Toolbox](windows.md#docker-toolbox)

## Docker for Windows

Docker for Windows是我们提供的最新的PC安装方式。它以一个原生Windows应用方式运行，使用Hyper-V来虚拟化Docker Engine环境和Docker进程对Linux 内核某型特性的要求。

参见[Getting Started with Docker for Windows](/docker-for-windows/)，了解下载、安装说明和所有Docker for Windows的相关内容。

**要求**

* 64bit Windows 10 Pro, Enterprise and Education (1511 November update, Build 10586 or later). 未来我们会支持更多版本的Windows 10.

* Hyper-V需要开启。在使用Docker for Windows过程中会将它开启（需要重启）。

## Docker Toolbox

如果你的Windows系统不满足安装Docker for Windows的要求，请使用<a href="https://www.docker.com/products/docker-toolbox" target="_blank">get Docker Toolbox</a>

参见[Docker Toolbox Overview](/toolbox/overview.md) 中的说明来安装Docker Toolbox

Docker Toolbox并不在Windows本机直接运行Docker，它使用 `docker-machine` 来创建并连接虚拟机（VM），Docker跑在这台Linux虚拟机上，虚拟机运行在你的Windows系统中。

**要求**

机器必须安装64位Windows7或更高版本才能运行Docker，此外还要保证机器已经开启了虚拟化功能。完整的安装说明请见[Toolbox install instructions for Mac](/toolbox/toolbox_install_mac.md).

## 更多内容

* 如果你是Docker新手，[入门指南](../getstarted/index.md)中包含使用Docker命令、运行容器、构建镜像、使用Docker Hub等一系列内容。

* 你可以在[Docker Engine User Guide](../userguide/index.md)中的[Learn by example](../tutorials/index.md)部分查看到更多示例。

* 如果你对Kitematic GUI有兴趣，可以参考[Kitematic user guide](/kitematic/userguide/).

> **注意**: The Boot2Docker 前几个版本中就已经不再维护，请使用Docker Machine或更新的Docker for Windows.