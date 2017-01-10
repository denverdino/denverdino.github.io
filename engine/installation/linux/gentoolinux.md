---
description: Installation instructions for Docker on Gentoo.
keywords: gentoo linux, docker, documentation,  installation
redirect_from:
- /engine/installation/gentoolinux/
title: Install Docker on Gentoo
---

在 Gentoo Linux 上安装 Docker 有两种方法：**官方安装** 方法和 `docker-overlay` 方法。
Gentoo 团队官方项目页[Gentoo Docker](https://wiki.gentoo.org/wiki/Project:Docker) team.

## 官方方式
如果你正在寻找一种稳定的方案，首推官方方式，可以直接在tree上安装官方 `app-emulation/docker` 包

[IRC](http://webchat.freenode.net?channels=%23gentoo-containers&uio=d4) channel on the Freenode network.
如果 ebuild 时出现任何问题，包括缺少内核配置标识或依赖，请到 Gentoo 的 [Bugzilla](https://bugs.gentoo.org)上提交问题，并指派给 `docker AT gentoo DOT org` 
，或者加入 Freenode 的 Gentoo 官方[IRC](http://webchat.freenode.net?channels=%23gentoo-containers&uio=d4)频道来提问。

## docker-overlay 方式

如果你正在寻找一个 `-bin` ebuild, live ebuild, 或者 bleeding edge ebuild，可以使用[docker-overlay](https://github.com/tianon/docker-overlay)，它使用 `app-portage/layman` 添加第三方portage。
最新的安装和使用overlay说明请查看[overlay](https://github.com/tianon/docker-overlay/blob/master/README.md#using-this-overlay)。

如果 ebuild 或者生成的二进制文件时出现任何问题，特别是缺少内核配置标识或依赖关系，
请在 `docker-overlay` 仓库提交一个[issue](https://github.com/tianon/docker-overlay/issues)或者直接在 Freenode 的 `#docker` IRC频道上联系 `tianon` 。

## 安装

### 可用的使用标识

| USE Flag      | Default | Description |
| ------------- |:-------:|:------------|
| aufs          |         |Enables dependencies for the "aufs" graph driver, including necessary kernel flags.|
| btrfs         |         |Enables dependencies for the "btrfs" graph driver, including necessary kernel flags.|
| contrib       |  Yes    |Install additional contributed scripts and components.|
| device-mapper |  Yes    |Enables dependencies for the "devicemapper" graph driver, including necessary kernel flags.|
| doc           |         |Add extra documentation, such as API and Javadoc. It is recommended to enable per package instead of globally.|
| vim-syntax    |         |Pulls in related vim syntax scripts.|
| zsh-completion|         |Enable zsh completion support.|

USE flags 详细介绍请见 [tianon's
blog](https://tianon.github.io/post/2014/05/17/docker-on-gentoo.html).

这个包会获取必要的依赖并提示需要的内核选项。

    $ sudo emerge -av app-emulation/docker

>注: 有时候官方的 **Gentoo tree** 和 **docker-overlay** 的最新版本不一致。
>请耐心等待，最新版本会很快更新。

## 启动 Docker

请确保您运行的内核包含了所有必要的模块和配置（根据你要使用的存储驱动，选择device-mapper，AUFS 或 Btrfs）。

使用 Docker，`docker` 进程必须以 **root**用户运行。
对于 **非root** 用户使用 Docker，可以使用下边的命令，将自己添加到 **docker** 用户组：

    $ sudo groupadd docker
    $ sudo usermod -a -G docker user

### OpenRC

启动 `docker` 进程:

    $ sudo /etc/init.d/docker start

开机启动:

    $ sudo rc-update add docker default

### systemd

启动 `docker` 进程:

    $ sudo systemctl start docker

开机启动:

    $ sudo systemctl enable docker

如果你想要添加一个 HTTP 代理，需要为 Docker 运行文件设置不同的目录或分区。
如果需要定制一些其它的功能，请阅读我们的systemd文章，了解如何[定制Docker进程](../../admin/systemd.md)

## 卸载

卸载 Docker 包:

    $ sudo emerge -cav app-emulation/docker

卸载 Docker 包及不再使用的相关依赖包:

    $ sudo emerge -C app-emulation/docker

上述命令不会删除主机上的镜像，容器，数据卷及用户配置文件。如果想删除镜像，容器，数据卷可以使用下面命令：

    $ rm -rf /var/lib/docker

用户配置文件需要手动查找删除。