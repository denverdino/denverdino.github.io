---
description: Installation instructions for Docker on openSUSE and on SUSE Linux Enterprise.
keywords: openSUSE, SUSE Linux Enterprise, SUSE, SLE, docker, documentation,  installation
redirect_from:
- /engine/installation/SUSE/
title: Install Docker on openSUSE and SUSE Linux Enterprise
---

这个文档提供了在openSUSE 和 SUSE 操作系统上安装和配置最新版Docker的说明.

>**说明:** 你可以从[开放构建服务](https://build.opensuse.org/)的[虚拟化:容器项目](https://build.opensuse.org/project/show/Virtualization:containers)
项目中找到目前维护的最前沿的Docker版本.
这个项目也提供和Docker生态系统相关的其他包(例如, Docker Compose).

## 先决条件

你必须运行在64位架构的系统上.

## openSUSE

从 13.2版本以后Docker就作为官方openSUSE库的一部分了.
在 openSUSE 13.2版本以后就不需要添加额外的库了.

## SUSE Linux 企业版

从SUSE Linux 企业版 12以及以后的开始，官方已经开始支持Docker. 你可以在`Container`模块找到最新支持的Docker软件包.
通过以下步骤来启动该模块:

1. 开始 YaST, 然后选择 *Software > Software Repositories*.
2. 点击 *Add* 按钮将打开一个对话框.
3. 选择 *Extensions and Module from Registration Server* 然后点击 *Next*.
4. 从可用的扩展和模块列表, 选择 *Container Module* 并且点击 *Next*.
   容器模块和它的资源库会被添加到你的系统中.
5. 你可以使用订阅管理工具从SMT服务器更新资源库.

否则运行以下的命令:

    $ sudo SUSEConnect -p sle-module-containers/12/x86_64 -r ''

    >**说明:** 目前 `-r ''` 参数用来避免`SUSEConnect`已知的一些限制. 


在[开放构建服务](https://build.opensuse.org/) 的[虚拟化:容器项目](https://build.opensuse.org/project/show/Virtualization:containers)
包含了SUSE Linux 企业版支持的一些最前沿的Docker软件包. 然后SUSE**不再支持**这些包.

### 安装 Docker

1. 安装Docker 软件包:

        $ sudo zypper in docker

2. 启动Docker.

        $ sudo systemctl start docker

3. 测试Docker安装.

        $ sudo docker run hello-world

## 配置Docker开机启动

你可以使用以下的步骤在openSUSE 或者 SUSE Linux 企业版上设置`docker daemon`开机启动,设置如下:

    $ sudo systemctl enable docker

`docker`包创建了一个`docker`的用户组.非`root`用户作为该组成员和Docker进程交互.
你可以使用一下的命令格式添加用户:

    $ sudo /usr/sbin/usermod -a -G docker <username>

一旦添加用户，确保他们重新登录获得新的权限.

## 允许访问外部网络

如果你想要你的容器可以访问外部网络, 你必须启用`net.ipv4.ip_forward`规则.
使用YaST来完成.

openSUSE Tumbleweed和以后的版本，打开**System -> Network Settings -> Routing**菜单.
SUSE Linux 企业版 12 和之前的 openSUSE 版本,打开**Network Devices -> Network Settings -> Routing**菜单
选中 *Enable IPv4 Forwarding* 复选框.

当网络是由Network Manager而不是YaST处理，你必须编辑`/etc/sysconfig/SuSEfirewall2`文件,
设置`FW_ROUTE`标识为`yes`，如下：

    FW_ROUTE="yes"

## 自定义进程

如果你想要添加一个 HTTP 代理，为 Docker 运行文件设置不同的目录或分区，又或者定制一些其它的功能，请阅读我们的系统文章，
了解[如何定制Docker进程](../../admin/systemd.md).

## 卸载

卸载Docker软件包:

    $ sudo zypper rm docker

以上并不会移除镜像，容器，数据卷以及用户创建的配置文件等.如果你想删除所有的镜像，容器，数据卷，可以
运行一下的命令:

    $ rm -rf /var/lib/docker

你必须手动删除用户创建的配置文件.

## 更多参考

你可以在SUSE网站找到更多在openSUSE 或 SUSE Linux 企业版上使用Docker的细节.
[Docker快速指南](https://www.suse.com/documentation/sles-12/dockerquick/data/dockerquick.html)
该文档的目标是SUSE Linux 企业版, 但是也适用于openSUSE.

更多的查看 [用户指南](../../userguide/index.md).
