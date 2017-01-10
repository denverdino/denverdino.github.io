---
description: Installing and running an SSHd service on Docker
keywords: docker, example, package installation,  networking
title: Dockerize an SSH service
---

## 构建一个 `eg_sshd` 镜像


下面的 `Dockerfile` 在容器中建立了一个 SSHd 服务，依靠该服务可以连接、查看其它容器的数据卷，也可以快速连接一个测试容器。

```Dockerfile
FROM ubuntu:16.04
MAINTAINER Sven Dowideit <SvenDowideit@docker.com>

RUN apt-get update && apt-get install -y openssh-server
RUN mkdir /var/run/sshd
RUN echo 'root:screencast' | chpasswd
RUN sed -i 's/PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config

# SSH登陆修复。否则用户会在登陆后被踢出
RUN sed 's@session\s*required\s*pam_loginuid.so@session optional pam_loginuid.so@g' -i /etc/pam.d/sshd

ENV NOTVISIBLE "in users profile"
RUN echo "export VISIBLE=now" >> /etc/profile

EXPOSE 22
CMD ["/usr/sbin/sshd", "-D"]
```

使用以下命令构建镜像:

```bash
$ docker build -t eg_sshd .
```

## 启动一个 `test_sshd` 容器

基于刚才的镜像启动容器. 您可以使用命令 `docker port` 查看容器的22端口暴露在宿主机的哪个端口上：

```bash
$ docker run -d -P --name test_sshd eg_sshd
$ docker port test_sshd 22

0.0.0.0:49154
```

现在您可以以 `root` 角色 ssh 到容器的IP地址上（您可以使用 `docker inspect` 来查看该信息）或者使用 Docker 后台进程所在的宿主机的IP地址（在宿主机上执行 `ip address` 或 `ifconfig` 可以获取到该IP）配合端口 `49154` 来登陆，如果您当前是在Docker后台进程的宿主机上的话，可以直接使用 `localhost` 来做 ssh 登陆：

```bash
$ ssh root@192.168.1.2 -p 49154
# The password is ``screencast``.
root@f38c87f2a42d:/#
```

## 环境变量
在普通的Docker 机制下，使用 `sshd` 后台进程新生成shell来传递环境变量给用户的shell会非常复杂，因为 `sshd` 会在启动shell之前消除环境变量。

如果您使用 `Dockerfile` 中的 `ENV` 来设置环境变量，则需要将这些变量更新到 shell 初始化文件中，例如上面的 `Dockerfile` 中展示的将变量值更新到 `/etc/profile` 文件的案例。

如果您想要传递命令 `docker run -e ENV=value` 中的环境变量值，您需要写一个小的脚本在您执行 `sshd -D` 命令之前来完成上面所提到的更新到shell初始化文件中的事情，然后使用这个脚本来替换原有的 `CMD`  配置。

## 清理

最后，在测试以后通过停止容器、移除容器、移除镜像来清理环境。

```bash
$ docker stop test_sshd
$ docker rm test_sshd
$ docker rmi eg_sshd
```
