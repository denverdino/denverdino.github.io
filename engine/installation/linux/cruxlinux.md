---
description: Docker installation on CRUX Linux.
keywords: crux linux, Docker, documentation,  installation
redirect_from:
- /engine/installation/cruxlinux/
title: Install Docker on CRUX Linux
---

在CRUX Linux 安装Docker可以通过官方的
 [contrib](http://crux.nu/portdb/?a=repo&q=contrib) ports:

- docker

`docker` port 将构建安装最新版本的Docker.


## 安装

如果你的版本允许，更新你的 ports 目录树并且安装docker:

    $ sudo prt-get depinst docker


## 内核要求

如果使 **CRUX+Docker** H主机正常工作，你必须确保你的内核安装必要的模块来保证 Docker 进程的正常运行.

请阅读 `README`:

    $ sudo prt-get readme docker

`docker` port 安装由Docker发行版提供的 `contrib/check-config.sh` 脚本,
以供检查你的内核配置是否合适安装Docker主机.

运行下面的命令来检查你的内核:

    $ /usr/share/docker/check-config.sh

## 启动Docker

我们提供一个 rc 脚本来创建 Docker.请使用下列命令来启动 Docker 服务:

    $ sudo /etc/rc.d/docker start

设置开机启动:

 - 编辑 `/etc/rc.conf`
 - 将 `docker` 放到 `SERVICES=(...)` 数组里的 `net` 之后.

## 镜像

`官方库`中提供了CRUX镜像。你可以使用`pull`命令在使用这个镜像,当然你也可以在
`Dockerfile(s)`文件的`FROM`部分设置使用.

    $ docker pull crux
    $ docker run -i -t crux

在Docker Hub中也有其他用户贡献的 [CRUX基础镜像](https://hub.docker.com/_/crux/).


## 卸载

卸载Docker软件包:

    $ sudo prt-get remove docker

以上的命令并不会移除镜像，容器，数据卷或者用户创建的配置文件。如果你想删除全部的镜像，容器以及数据卷
运行如下的命令：

    $ rm -rf /var/lib/docker

你必须手动删除用户配置文件.

## Issues

如果你有任何的问题，欢迎提到
[CRUX Bug Tracker](http://crux.nu/bugs/).

## 支持

需要帮忙请联系 [CRUX Mailing List](http://crux.nu/Main/MailingLists)
或者加入 CRUX's [IRC Channels](http://crux.nu/Main/IrcChannels). 通过
[FreeNode](http://freenode.net/) IRC Network.
