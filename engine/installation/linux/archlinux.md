---
description: Installation instructions for Docker on ArchLinux.
keywords: arch linux, docker, documentation,  installation
redirect_from:
- /engine/installation/archlinux/
title: Install Docker on Arch Linux
---

你可以使用Arch Linux社区发布的软件包进行安装:

 - [docker](https://www.archlinux.org/packages/community/x86_64/docker/)

或者使用 AUR 包:

 - [docker-git](https://aur.archlinux.org/packages/docker-git/)

docker软件包将会安装最新版本的Docker. docker-git 则是由master 分支构建的包.

## 依赖

Docker 依赖于几个指定的安装包. 核心的依赖包如下:

 - bridge-utils
 - device-mapper
 - iproute2
 - sqlite

## 安装

对于一般包简单安装:

    $ sudo pacman -S docker

这样就安装了所需要的一切.

对于AUR包执行:

    $ yaourt -S docker-git

安装说明假设已安装好 **yaourt** . 如果你之前没有安装构建过这个包，请参考 [Arch User
Repository](https://wiki.archlinux.org/index.php/Arch_User_Repository#Installing_packages)


## 启动 Docker

Docker 会创建一个系统服务，用下面命令来启动 Docker:

    $ sudo systemctl start docker

设置开机启动:

    $ sudo systemctl enable docker

## 自定义进程选项

如果你想要添加一个 HTTP 代理，为 Docker 运行文件设置不同的目录或分区，又或者定制一些其它的功能，请阅读我们的系统文章，
了解[如何定制Docker进程](../../admin/systemd.md).

## 通过手动定义网络运行Docker

如果你手动配置网络通过`systemd-network` 220 或者更高的版本，你使用Docker启动的容器将不能访问你的网络。
从220版本开始，对于特定网络的转发规则(`net.ipv4.conf.<interface>.forwarding`) 默认值为*off*.
该设置阻止IP转发。该规则与Docker在容器中开启`net.ipv4.conf.all.forwarding`规则是冲突的。

为了使得以上可以工作, 需要在你的Docker宿主机下，编辑`/etc/systemd/network/`目录下的 `<interface>.network`文件。
添加如下内容:

```
[Network]
...
IPForward=kernel
...
```

这个配置允许从容器中进行IP转发.

## 卸载

卸载Docker软件包:

    $ sudo pacman -R docker

卸载Docker 以及依赖的软件包:

    $ sudo pacman -Rns docker

以上的命令不会移除镜像，容器，数据卷以及用户创建的配置文件。
如果想删除全部的镜像，容器，数据卷，运行以下命令：

    $ rm -rf /var/lib/docker

你必须手动删除用户创建的配置文件。
