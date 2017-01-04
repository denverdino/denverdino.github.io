---
description: How to create base images
keywords: Examples, Usage, base image, docker, documentation,  examples
redirect_from:
- /engine/articles/baseimages/
title: Create a base image
---

你开始希望创建自己的[*基础镜像*](../../reference/glossary.md#base-image)了吗? 太棒了！

具体的制作过程将依赖于你希望打包的特定的Linux发行版。下面，我们有几个镜像制作的示例，
欢迎通过提交pull请求来贡献更多的示例。

## 使用归档文件来创建一个完整的镜像

通常来说，当你需要打包某一Linux的发行版，将其作为基础镜像时，你需要一个运行该发行版的机器，
尽管对于某些工具来说这不是必须的，例如Debian的[Debootstrap](https://wiki.debian.org/Debootstrap),
你可以用它来构建Ubuntu的镜像。

通过如下步骤，制作一个Ubuntu的基础镜像可以很简单：
    $ sudo debootstrap raring raring > /dev/null
    $ sudo tar -C raring -c . | docker import - raring

    a29c15f1bf7a

    $ docker run raring cat /etc/lsb-release

    DISTRIB_ID=Ubuntu
    DISTRIB_RELEASE=13.04
    DISTRIB_CODENAME=raring
    DISTRIB_DESCRIPTION="Ubuntu 13.04"

在Docker GitHub 仓库中，还有更多的示例脚本用于创建基础镜像：

 - [BusyBox](https://github.com/docker/docker/blob/master/contrib/mkimage-busybox.sh)
 - CentOS / Scientific Linux CERN (SLC) [在 Debian/Ubuntu上](
   https://github.com/docker/docker/blob/master/contrib/mkimage-rinse.sh) 或者
   [在 CentOS/RHEL/SLC/等上](
   https://github.com/docker/docker/blob/master/contrib/mkimage-yum.sh)
 - [Debian / Ubuntu](
   https://github.com/docker/docker/blob/master/contrib/mkimage-debootstrap.sh)

## 使用scratch创建一个简单的基础镜像

你可以使用一个Docker保留的最小化的镜像，`scratch`，作为构建容器的起点。使用`scratch`镜像告诉了构建过程你希望在`Dockerfile`中的下一条命令作为镜像的第一个文件系统层。
尽管`scratch`镜像出现在Docker hub的仓库中，你不能pull它，执行它，或者给其他的镜像打上`scratch`的标签。你能做的就是在`Dockerfile`中引用该镜像，例如使用`scratch`创建一个最小的容器：

    FROM scratch
    ADD hello /
    CMD ["/hello"]

这一示例创建了在教材中使用的hello-world镜像。
如果你想测试这一镜像的制作，你可以克隆[镜像仓库](https://github.com/docker-library/hello-world)


## 更多资源

还有更多的资源来帮助你编写`Dockerfile`.

* 在引用部分，对于Dockerfile出现的所有指令，有一个完整的[说明](../../reference/builder.md)
* 为了帮助您编写清晰，可读，维护性好的`Dockerfile`，我们提供了[`Dockerfile`最佳实践指南](dockerfile_best-practices.md)
* 如果你的目标是创建一个新的官方仓库，请务必阅读[官方仓库](/docker-hub/official_repos/)。
