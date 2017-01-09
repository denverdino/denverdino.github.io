---
description: How to use Supervisor process management with Docker
keywords: docker, supervisor,  process management
redirect_from:
- /engine/articles/using_supervisord/
title: Use Supervisor with Docker
---

> **注意**:
> - **如果你讨厌sudo** 请看 [*非管理员
>   权限*](../installation/binaries.md#giving-non-root-access)


传统上，容器在启动时运行单个进程例如Apache守护进程或SSH服务器守护进程。通常，当你想运行在容器中有多个进程， 你有很多方法可以选择，从CMD指定Bash到安装一个京城管理工具。

在本例中，您将使用过程管理工具[Supervisor](http://supervisord.org/)来管理容器中的多个进程。使用Supervisor能更好的控制，管理和重启容器中多进程。 为了演示这个，我们要安装并管理SSH守护程序和Apache守护程序。

## 创建Dockerfile

让我们开始为我们的新容器镜像创建一个基本的“Dockerfile”


```Dockerfile
FROM ubuntu:16.04
MAINTAINER examples@docker.com
```

## 安装 Supervisor

您需要在容器中安装SSH， Apache和Supervisor。

```Dockerfile
RUN apt-get update && apt-get install -y openssh-server apache2 supervisor
RUN mkdir -p /var/lock/apache2 /var/run/apache2 /var/run/sshd /var/log/supervisor
```


第一个`RUN`指令安装`openssh-server`，`apache2`和`supervisor`（提供Supervisor守护程序）包。另一个“RUN”
指令创建运行SSH守护程序所需的四个新目录和Supervisor。

## 添加 Supervisor的配置文件

现在让我们为Supervisor添加一个配置文件。将调用默认文件`supervisord.conf`，位于`/etc/supervisor/conf.d /`。


```Dockerfile
COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf
```

查看`supervisord.conf`文件

```ini
[supervisord]
nodaemon=true

[program:sshd]
command=/usr/sbin/sshd -D

[program:apache2]
command=/bin/bash -c "source /etc/apache2/envvars && exec /usr/sbin/apache2 -DFOREGROUND"
```

`supervisord.conf'配置文件包含配置Supervisor的指令及其管理的进程。第一段`[supervisord]`指定自身的配置。使用`nodaemon`指令告诉Supervisor交互运行，而不是守护进程。

后两段表明管理我们要控制的服务。每段控制一个单独的进程。每段都包含一个`command`,这个`command`用来启动收管理的进程。


## 公开端口和运行 Supervisor

最后我们公开端口`22`和`80`，和指定容器启动时的`CMD`---- Supervisor。

```Dockerfile
EXPOSE 22 80
CMD ["/usr/bin/supervisord"]
```

以上指令告诉容器公开端口22和80，并且指定`/usr/bin/supervisord`二进制应该在容器启动时执行。


## 构建容器镜像

完整的Dockerfile:

```Dockerfile
FROM ubuntu:16.04
MAINTAINER examples@docker.com

RUN apt-get update && apt-get install -y openssh-server apache2 supervisor
RUN mkdir -p /var/lock/apache2 /var/run/apache2 /var/run/sshd /var/log/supervisor

COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf

EXPOSE 22 80
CMD ["/usr/bin/supervisord"]
```

并且`supervisord.conf`文件应该如下所示;

```ini
[supervisord]
nodaemon=true

[program:sshd]
command=/usr/sbin/sshd -D

[program:apache2]
command=/bin/bash -c "source /etc/apache2/envvars && exec /usr/sbin/apache2 -DFOREGROUND"
```

你应该用下面的命令构建;

```bash
$ docker build -t mysupervisord .
```

## 运行Supervisor 容器

一旦镜像创建成功，就可以创建容器了。

```bash
$ docker run -p 22 -p 80 -t -i mysupervisord
2013-11-25 18:53:22,312 CRIT Supervisor running as root (no user in config file)
2013-11-25 18:53:22,312 WARN Included extra file "/etc/supervisor/conf.d/supervisord.conf" during parsing
2013-11-25 18:53:22,342 INFO supervisord started with pid 1
2013-11-25 18:53:23,346 INFO spawned: 'sshd' with pid 6
2013-11-25 18:53:23,349 INFO spawned: 'apache2' with pid 7
...
```


您使用`docker run`命令交互式启动了一个新容器。该容器运行Supervisor并启动SSH和Apache守护进程。我们已经指定了`-p'标志来显示端口22和80.我们可以从
公开标识的端口访问SSH和Apache。
