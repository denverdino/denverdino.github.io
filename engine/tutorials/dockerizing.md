---
description: 一个简单的'Hello world练习，介绍了Docker。
keywords: docker指南，docker，docker平台，如何，dockerize，dockerizing应用程序，dockerizing应用程序，容器，容器
redirect_from: 
- /engine/userguide/containers/dockerizing/
- /engine/userguide/dockerizing/
title: 容器中运行Hello World
---

*那么这个Docker到底是什么呢？*

Docker允许您在容器内部运行应用程序，创建您自己的容器世界。
在容器中运行应用程序需要一个命令:`docker run`。

> **Note**:根据您的Docker系统配置，您可能需要为这个页面上的所有`docker`命令加一个`sudo`前缀。为了避免这个加`sudo`的操作，系统管理员可以创建一个名为`docker`的Unix用户组，并将当前用户添加进去。

## 运行Hello World

让我们运行一个hello world容器。

	$ docker run ubuntu /bin/echo 'Hello world'

	Hello world

您刚刚启动了您的第一个容器！

在这个例子中:

* `docker run`运行了一个容器。

* `ubuntu`是您运行的镜像。当您指定一个镜像时，Docker首先查看Docker主机上的镜像存储。如果镜像不存在于本地，则从公共镜像仓库[Docker Hub](https://hub.docker.com)中拉取（pull）。

* `/bin/echo`是在新容器中运行的命令。

现在容器启动了。 Docker创建了一个新的Ubuntu系统环境，并在其中执行`/bin/echo`命令，然后打印出来:

	Hello world

那么，容器启动之后发生了什么？好了，Docker容器只有在您指定的命令是活跃的情况下才会一直运行。因此，在上述示例中，一旦命令被执行完成，容器就停止并退出了。

## 运行交互式容器

让我们在容器中运行一个新命令。

	$ docker run -t -i ubuntu / bin / bash

	root@af8bae53bdd3:/#

在这个例子中:

* `docker run`运行一个容器。
* `ubuntu`是您想运行的镜像。
* `-t`标志在新容器内分配一个伪tty设备或终端。
* `-i`标志允许您通过抓取容器的标准输入（`STDIN`）来进行交互式连接。
* `/bin/bash`在我们的容器中启动了一个Bash shell。

容器启动。我们可以看到里面有一个命令提示符:

	root@af8bae53bdd3:/#

让我们尝试在容器中运行一些命令:

	root@af8bae53bdd3:/#pwd

	/

	root@af8bae53bdd3:/#ls

	bin boot dev etc home lib lib64 media mnt opt proc root run sbin srv sys tmp usr var

在这个例子中:

* `pwd`显示当前目录，`/`根目录。
* `ls`显示典型Linux文件系统的根目录的目录列表。

现在，您可以在这个容器里面自由发挥。完成后，运行`exit`命令或输入Ctrl-D退出交互式shell。

	root@af8bae53bdd3:/#exit

> **注意:**与我们以前的容器一样，Bash shell进程完成后，容器停止。

## 启动Hello world的守护进程

让我们创建一个作为守护进程运行的容器。

	$ docker run -d ubuntu /bin/sh -c“while true; do echo hello world; sleep 1; done”

	1e5535038e285177d5214659a068137486f96ee5c2e85a4ac52dc83f2ebe4147

在这个例子中:

* `docker run`运行容器。
* `-d`标志在后台运行容器（守护进程化）。
* `ubuntu`是您想运行的镜像。

最后，我们指定要运行的命令:

	/bin/sh -c "while true; do echo hello world; sleep 1; done"


在输出中，我们看不到“hello world”而是一个长字符串:

	1e5535038e285177d5214659a068137486f96ee5c2e85a4ac52dc83f2ebe4147

这个长字符串称为*容器ID*。它唯一地标识一个容器，所以我们可以使用它。

> **注意:**
> 容器ID有点长，笨重。后来，我们将介绍短ID和更容易使用的方式来命名我们的容器。

我们可以使用这个容器ID来看看我们的`hello world`守护进程正在发生什么。

首先，让我们确保我们的容器正在运行。运行`docker ps`命令。
`docker ps`命令向Docker daemon守护进程查询所有容器的列表信息。

	$ docker ps

	CONTAINER ID IMAGE COMMAND CREATED STATUS PORTS NAMES
	1e5535038e28 ubuntu /bin/sh -c'while tr 2分钟前上传1分钟insane_babbage

在这个例子中，我们可以看到我们的守护进程容器。 `docker ps`返回一些有用的信息:

* `1e5535038e28`是容器ID的较短变体。
* `ubuntu`是使用的映像。
* 命令，状态和分配的名称`insane_babbage`。


> **注意:**
> Docker自动为任何已启动的容器生成名称。
> 我们稍后会看到如何自己指定容器的名字。

现在，我们知道容器正在运行。但是它是正在做我们要求它做的工作吗？要看到这个，我们将使用`docker logs`命令查看容器内部标准输出。

让我们使用容器名称`insane_babbage`。

    $ docker logs insane_babbage

    hello world
    hello world
    hello world
    . . .

在这个例子中:

* `docker logs`查看容器内部并返回`hello world`。

很好！容器工作正常，您刚刚创建了第一个Docker化应用程序！

接下来，运行`docker stop`命令来停止我们的刚刚创建的容器。

	$ docker stop insane_babbage

	insane_babbage

`docker stop`命令告诉Docker优雅的停止正在运行的容器，并返回它正在停止的容器的名称。

让我们通过`docker ps`检查`docker stop`命令是否起作用了。

	$ docker ps

	CONTAINER ID IMAGE COMMAND CREATED STATUS PORTS NAMES

Excellent。我们的容器停止了。

# 下一步

到目前为止，您使用`docker run`命令启动了您的第一个容器。您运行了一个在前台运行的*交互式容器*。您还运行了一个在后台运行*detached容器*。在此过程中，您了解了以下几个Docker命令:

* `docker ps` - 列出容器。
* `docker logs` - 显示容器的标准输出。
* `docker stop` - 停止运行容器。

现在，您可以了解有关Docker的更多信息以及如何执行一些更高级的任务。转到["*运行一个简单的应用程序*"](usingdocker.md)来用Docker客户端构建一个真实的web应用程序。