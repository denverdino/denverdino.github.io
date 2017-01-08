---
description: Installing and running an apt-cacher-ng service
keywords: docker, example, package installation, networking, debian,  ubuntu
title: 容器内运行一个 apt-cacher-ng 服务
---

> **Note**:
> - **如果你不喜欢sudo** 你可以看 [*Giving non-root
>   access*](../installation/binaries.md#giving-non-root-access).
> - **如果你用 macOS 或者 docker via TCP** 那么你不应该用
>   sudo。

当你有多台Docker服务，或者在一台容器上构建不相关的Docker容器，以至于无法使用Docker build的缓存。它可以被用于对你的安装包做一个缓存代理，对任何第二次下载的镜像所启动的容器，可以做到几乎瞬间启动。

用以下的Dockerfile:

    #
    # Build: docker build -t apt-cacher .
    # Run: docker run -d -p 3142:3142 --name apt-cacher-run apt-cacher
    #
    # and then you can run containers with:
    #   docker run -t -i --rm -e http_proxy http://dockerhost:3142/ debian bash
    #
    # Here, `dockerhost` is the IP address or FQDN of a host running the Docker daemon
    # which acts as an APT proxy server.
    FROM        ubuntu
    MAINTAINER  SvenDowideit@docker.com

    VOLUME      ["/var/cache/apt-cacher-ng"]
    RUN     apt-get update && apt-get install -y apt-cacher-ng

    EXPOSE      3142
    CMD     chmod 777 /var/cache/apt-cacher-ng && /etc/init.d/apt-cacher-ng start && tail -f /var/log/apt-cacher-ng/*

用命令构建Docker镜像:

    $ docker build -t eg_apt_cacher_ng .

然后运行它，对暴露的端口做一个宿主机上的映射：

    $ docker run -d -p 3142:3142 --name test_apt_cacher_ng eg_apt_cacher_ng

如果要看日志文件的`tailed`模式，你可以用：

    $ docker logs -f test_apt_cacher_ng

如果你的Debian基础容器想接入代理，你需要执行以下步骤。注意，你必须用运行`test_apt_cacher_ng`的主机的IP地址或者完整域名，替换`dockerhost`。 

1. 增加一个 apt Proxy 配置
   `echo 'Acquire::http { Proxy "http://dockerhost:3142"; };' >> /etc/apt/conf.d/01proxy`
2. 设置一个环境变量：
   `http_proxy=http://dockerhost:3142/`
3. 修改你的 `sources.list` 条目，开头是
   `http://dockerhost:3142/`
4. 用 `--link`参数，连接Debian基础容器到APT代理容器。
5. 为你的Debian基础容器和APT代理容器。

**步骤 1** 通过在公共的base目录下，注入一个本地版本的配置文件，引入我们的配置：

    FROM ubuntu
    RUN  echo 'Acquire::http { Proxy "http://dockerhost:3142"; };' >> /etc/apt/apt.conf.d/01proxy
    RUN apt-get update && apt-get install -y vim git

    # docker build -t my_ubuntu .

**步骤 2** 很容易测试。但是会使其他使用了`http_proxy`变量的HTTP客户端失效，比如 `curl`, `wget` 等等：

    $ docker run --rm -t -i -e http_proxy=http://dockerhost:3142/ debian bash

**步骤 3** 是最容易的（least portable），但是需要一点时间。你也可以通过`Dockerfile` 执行它。 

**步骤 4** 使用以下命令，将Debian基础容器 连接到代理服务上:

    $ docker run -i -t --link test_apt_cacher_ng:apt_proxy -e http_proxy=http://apt_proxy:3142/ debian bash

**步骤 5** 为你的Debian基础容器和APT代理容器：

    $ docker network create mynetwork
    $ docker run -d -p 3142:3142 --network=mynetwork --name test_apt_cacher_ng eg_apt_cacher_ng
    $ docker run --rm -it --network=mynetwork -e http_proxy=http://test_apt_cacher_ng:3142/ debian bash

Apt-cacher-ng有一些工具，可以帮助你管理你的仓库， 通过`VOLUME`指令和我们构建服务的镜像，我们可以使用他们：

    $ docker run --rm -t -i --volumes-from test_apt_cacher_ng eg_apt_cacher_ng bash

    root@f38c87f2a42d:/# /usr/lib/apt-cacher-ng/distkill.pl
    Scanning /var/cache/apt-cacher-ng, please wait...
    Found distributions:
    bla, taggedcount: 0
         1. precise-security (36 index files)
         2. wheezy (25 index files)
         3. precise-updates (36 index files)
         4. precise (36 index files)
         5. wheezy-updates (18 index files)

    Found architectures:
         6. amd64 (36 index files)
         7. i386 (24 index files)

    WARNING: The removal action may wipe out whole directories containing
             index files. Select d to see detailed list.

    (Number nn: tag distribution or architecture nn; 0: exit; d: show details; r: remove tagged; q: quit): q

最后，清理你的测试： 停止并且移除你的容器，然后移除镜像。 

    $ docker stop test_apt_cacher_ng
    $ docker rm test_apt_cacher_ng
    $ docker rmi eg_apt_cacher_ng
