---
description: Instructions for installing Docker on Debian.
keywords: Docker, Docker documentation, installation,  debian
redirect_from:
- /engine/installation/debian/
title: Install Docker on Debian
---

以下的Debian版本支持Docker:

 - [*Debian testing stretch (64-bit)*](#debian-wheezy-stable-7-x-64-bit)
 - [*Debian 8.0 Jessie (64-bit)*](#debian-jessie-80-64-bit)
 - [*Debian 7.7 Wheezy (64-bit)*](#debian-wheezy-stable-7-x-64-bit) (backports required)

 >**提示**: 如果你之前已使用`APT`安装了Docker, 请确保更新你的`APT`源到最新的资源库.

## 先决条件

 Docker需要安装在一个64位的Debian系统上.
 另外，你的系统的内核至少需要3.10. 3.10小版本或者较新的维护版本仍然是可以被接受的。

 在低于3.10版本的内核上运行 Docker 会丢失一部分功能。在这些旧的版本上运行 Docker 会出现一些BUG，
 这些BUG在一定的条件里会导致数据的丢失，或者报一些严重的错误.

 检查当前系统的内核版本，打开命令行终端并且执行 `uname -r`命令来显示你的系统的内核版本:

     $ uname -r

 另外, 对Debian Wheezy的用户来说, backports必须是可用的. 以下是如何启动 backports 的方式在 Wheezy:

 1. 登陆到你的机器，以`sudo` 或者 `root` 权限打开终端.

 2. 使用你喜欢的编辑器打开文件 `/etc/apt/sources.list.d/backports.list` .

     如果不存在创建它.

 3. 移除存在的源.

 4. 添加backports 源从Debian Wheezy.

     示例源:

         deb http://http.debian.net/debian wheezy-backports main

 5. 更新软件包:

         $ apt-get update

### Update your apt repository

Docker的`APT` 仓库包含了Docker 1.7.1以及更高的版本。设置`APT`使用最新的库：

 1. 如果你未完成设置, 以 `sudo` 或者 `root` 身份登录到你的机器.

 2. 打开一个终端窗口.

 3. 清理旧的资源库.

         $ apt-get purge "lxc-docker*"
         $ apt-get purge "docker.io*"

 4. 更新软件包, 确保`APT`是以 `https` 方式进行, 且安装了CA证书.

         $ apt-get update
         $ apt-get install apt-transport-https ca-certificates

 5. 添加 `GPG` 秘钥.

         $ apt-key adv --keyserver hkp://p80.pool.sks-keyservers.net:80 --recv-keys 58118E89F3A912897C070ADBF76221572C52609D

 6. 使用你喜欢的编辑器打开文件 `/etc/apt/sources.list.d/docker.list` .

     如果不存在创建它.

 7. 移除存在的源..

 8. 为你的Debian 添加源.

     可用的源有:

    - On Debian Wheezy

            deb https://apt.dockerproject.org/repo debian-wheezy main

    - On Debian Jessie

            deb https://apt.dockerproject.org/repo debian-jessie main

    - On Debian Stretch/Sid

            deb https://apt.dockerproject.org/repo debian-stretch main

    > **说明**: Docker并没有提供所有架构的软件包. 要安装Docker到一个多架构系统上，
    > 在入口添加 `[arch=...]` 语句. 更多细节请参考
    > [Debian Multiarch wiki](https://wiki.debian.org/Multiarch/HOWTO#Setting_up_apt_sources)

 9. 保存并关闭文件.

 10. 更新 `APT` 包的索引.

         $ apt-get update

 11. 校验 `APT` 是从正确的仓库拉取镜像.

         $ apt-cache policy docker-engine

     自此以后，当你运行 `apt-get upgrade`, `APT`将拉取镜像从最新的源.

## Install Docker

安装Docker前, 确保你的 `APT` 源像先决条件里描述的那样被正确的设置.

1. 更新 `APT` 索引.

        $ sudo apt-get update

2. 安装Docker.

        $ sudo apt-get install docker-engine

3. 启动 `docker` 守护进程.

        $ sudo service docker start

4. 验证 `docker` 安装成功.

        $ sudo docker run hello-world

    该命令下载一个测试镜像且运行在容器里。当容器运行起来后，会打印一条信息，然后，退出。    


## 使用非roor

`docker`守护进程是以`root`用户身份运行，并且 `docker`守护进程使用Unix socket监听来
替代TCP端口监听. 默认情况下，Unix socket 属于 `root`用户, 当然其他用户也可以通过 `sudo`方式来访问.

如果你 (或者你安装Docker的时候) 创建一个叫 `docker`的用户组，并且为用户组添加用户.
这时候，当`docker` 守护进程启动的时候, `docker` 用户组对Unxi Socket有了读/写权限.
你必须使用root用户来运行`docker`守护进程, 但是你可以使用`docker`群组用户来使用
`docker` 客户端，你在使用 `docker` 命令的时候前边就不需要添加 `sudo`了。 从 Docker 0.9.0 
版本开始，你可以使用`-G` 来指定用户组.

> **警告**:
> `docker` 用户组 (或者用 `-G` 指定的用户组) 等同于
> `root`用户的权限; 有关安全影响的细节，请查看 [*Docker进程表面攻击细节*](../../security/security.md#docker-daemon-attack-surface).

**示例:**

    # Add the docker group if it doesn't already exist.
    $ sudo groupadd docker

    # Add the connected user "${USER}" to the docker group.
    # Change the user name to match your preferred user.
    # You may have to logout and log back in again for
    # this to take effect.
    $ sudo gpasswd -a ${USER} docker

    # Restart the Docker daemon.
    $ sudo service docker restart

## 升级 Docker

通过`apt-get`安装最新版本的 Docker :

    $ apt-get upgrade docker-engine

## 卸载

卸载 Docker 软件包:

    $ sudo apt-get purge docker-engine

卸载Docker以及依赖的软件包:

    $ sudo apt-get autoremove --purge docker-engine

以上的命令并不会移除镜像，容器，数据卷或者用户创建的配置文件。如果你想删除全部的镜像，容器以及数据卷
运行如下的命令：

    $ rm -rf /var/lib/docker

你必须手动删除用户配置文件.

## 下一步?

继续阅读 [用户指南](../../userguide/index.md).
