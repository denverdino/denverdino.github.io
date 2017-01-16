---
redirect_from:
- /engine/userguide/containers/usingdocker/
description: Learn how to manage and operate Docker containers.
keywords: docker, the docker guide, documentation, docker.io, monitoring containers, docker top, docker inspect, docker port, ports, docker logs, log, logs
menu:
  main:
    parent: engine_learn_menu
    weight: -5
title: Run a simple application
---

在["*Hello world in a container*"](dockerizing.md)中，您使用`docker run`命令启动了第一个容器。您运行了一个在前台运行的*交互式容器*。您还运行了一个*dettached的后台容器*。在此过程中，您了解了几个Docker命令：

* `docker ps` - 列出容器。
* `docker logs` - 显示容器的标准输出。
* `docker stop` - 停止运行容器。

## 了解Docker客户端

`docker`程序称为Docker客户端。客户端可以使用的每个操作都是一个命令，每个命令可以使用一系列标志和参数。

	# 用法：[sudo] docker [subcommand] [flags] [arguments] ..
	# 示例：
	$ docker run -i -t ubuntu /bin/bash

您可以通过使用`docker version`命令来返回当前安装的Docker客户端和守护进程的版本信息。

	$ docker version

此命令将不仅提供您正在使用的Docker客户端和守护进程的版本，而且还提供Go的版本(为Docker提供支持的编程语言)。

    Client:
     Version:      1.12.2
     API version:  1.24
     Go version:   go1.6.3
     Git commit:   bb80604
     Built:        Tue Oct 11 17:00:50 2016
     OS/Arch:      windows/amd64

    Server:
     Version:      1.12.3
     API version:  1.24
     Go version:   go1.6.3
     Git commit:   6b644ec
     Built:        Wed Oct 26 23:26:11 2016
     OS/Arch:      linux/amd64

## 获取Docker命令帮助

您可以显示特定Docker命令的帮助信息。帮助信息详细介绍了命令的选项及其用法。要查看所有可能的命令的列表，请使用以下命令：
	
	$ docker --help

要查看特定命令的用法，请使用`--help`标志指定命令：

	$ docker attach --help

    Usage: docker attach [OPTIONS] CONTAINER

    Attach to a running container

    Options:
      --detach-keys string   Override the key sequence for detaching a container
      --help                 Print usage
      --no-stdin             Do not attach STDIN
      --sig-proxy            Proxy all received signals to the process (default true)
 
> **注意：**
> 有关每个命令的更多详细信息和示例，请参阅本指南中的[命令参考](../reference/commandline/cli.md)。

## 在Docker中运行Web应用程序

现在您已经了解了一些有关Docker客户端的信息，您可以开始了解更重要的东西：运行更多的容器。到目前为止，您运行的任何容器都没有什么特别有用的，所以您可以通过在Docker中运行一个示例Web应用程序来改变这种状况。

对于Web应用程序，您将运行一个Python Flask应用程序。从`docker run`命令开始。

    $ docker run -d -P training/webapp python app.py

此命令由以下部分组成：

* `-d`标志在后台运行容器(作为所谓的守护进程)。
* `-P`标志将容器中任何所需的网络端口映射到您的主机。这使您可以浏览Web应用程序。
* `training/webapp`镜像是一个预构建的图像，包含一个简单的Python Flask网络应用程序。
* 剩余的参数组成在其中运行的命令容器。 `python app.py`命令启动Web应用程序。

> **注意：**
> 您可以在[command reference](../reference/commandline/run.md)和[docker run reference](../reference/run.md)中的`docker run`命令中查看更多详细信息。

## 查看Web应用程序容器

现在您可以使用`docker ps`命令看到您运行的容器。

    $ docker ps -l

    CONTAINER ID  IMAGE                   COMMAND       CREATED        STATUS        PORTS                    NAMES
    bc533791f3f5  training/webapp:latest  python app.py 5 seconds ago  Up 2 seconds  0.0.0.0:49155->5000/tcp  nostalgic_morse

`-l`标志只显示*最近运行的*容器的详细信息。

> **注意：**
>默认情况下，docker ps命令只显示有关运行容器的信息。如果您也想看到停止的容器，使用`-a`标志。

您可以看到您在[当您第一次docker化一个容器](dockerizing.md)中看到的相同的细节信息，并且添加了一个额外的重要的列`PORTS`。

    PORTS
    0.0.0.0:49155->5000/tcp

当您将`-P`标志传递给docker run命令时，Docker将容器中暴露的任何端口映射到您的主机。

> **注意：**
>您将学习更多关于如何在Docker镜像中暴露更多端口[参见](dockerimages.md)。

在这种情况下，Docker在端口49155上暴露了端口5000(默认的Python Flask端口)。

网络端口绑定在Docker中非常容易配置。在最后一个示例中，`-P`标志是`-p 5000`的快捷方式，它将容器内的端口5000映射到本地Docker主机上的高端端口(从*临时端口范围*，通常范围从32768到61000) 。您还可以使用`-p`标志将Docker容器绑定到特定端口，例如：

    $ docker run -d -p 80:5000 training/webapp python app.py

这将映射您的容器内的端口5000到本地主机上的端口80。您现在可能会问：为什么我们不想总是在Docker容器中使用1：1端口映射而选择使用映射到高端端口？好吧，1：1映射有一个约束，它只能映射本地主机上某个端口一次。

假设您想测试两个Python应用程序：都绑定到容器里面5000端口上。没有Docker的端口映射，您只能在Docker主机上一次访问一个。

因此，您现在可以在Web浏览器中浏览到端口49155查看应用程序。

![正在运行的Web应用程序的屏幕截图](webapp1.png)。

您的Python Web应用程序正常运行了！

> **注意：**
>如果您在macOS，Windows或Linux上使用虚拟机，
>您需要获取虚拟主机的IP，而不是使用localhost。
>您可以通过从命令行运行`docker-machine ip`来做到这一点：
>
> $ docker-machine ip
> 192.168.99.100
>
>在这种情况下，您将浏览到上面的例子中的“http://192.168.99.100:49155”。

## 快捷的网络端口映射

使用`docker ps`命令返回映射的端口有点笨拙，所以Docker有一个有用的快捷方式可以使用：`docker port`。要使用`docker port`，请指定您的容器的ID或名称，然后指定相应的需要暴露出去的端口。

	$ docker port nostalgic_morse 5000

	0.0.0.0:49155

在这种情况下，您也同时找到了映射到容器内端口5000的外部端口49155。

##查看Web应用程序的日志

您还可以更多地了解您的应用程序发生了什么，并使用您学到的另一个命令，`docker logs`。

    $ docker logs -f nostalgic_morse

    * Running on http://0.0.0.0:5000/
    10.0.2.2 - - [06/Nov/2016 20:16:31] "GET / HTTP/1.1" 200 -
    10.0.2.2 - - [06/Nov/2016 20:16:31] "GET /favicon.ico HTTP/1.1" 404 -

`-f`标志使`docker logs`命令像`tail -f`命令，并监视容器的标准输出。您可以在这里看到来自Flask的日志，它显示了在端口5000上运行的应用程序和它的访问日志信息。

## 查看Web应用程序容器的进程

除了容器的日志，您还可以使用`docker top`命令来检查容器中运行的进程。

    $ docker top nostalgic_morse

    PID                 USER                COMMAND
    854                 root                python app.py

在这里您可以看到`python app.py`命令是在容器中运行的唯一进程。

## Inspect Web应用容器

最后，可以使用`docker inspect`命令对Docker容器进行低级别的信息挖掘。它返回一个JSON文档，其中包含指定容器的相关配置和状态信息。

	$ docker inspect nostalgic_morse

您可以看到该JSON输出的示例。

    [{
        "ID": "bc533791f3f500b280a9626688bc79e342e3ea0d528efe3a86a51ecb28ea20",
        "Created": "2014-05-26T05:52:40.808952951Z",
        "Path": "python",
        "Args": [
           "app.py"
        ],
        "Config": {
           "Hostname": "bc533791f3f5",
           "Domainname": "",
           "User": "",
    . . .

您还可以通过请求特定元素来缩小要返回的信息，例如返回容器的IP地址，使用如下命令：


    {% raw %}
    $ docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' nostalgic_morse
    {% endraw %}

    172.17.0.5

## 停止Web应用容器

Web应用程序仍在容器内运行。您可以使用`docker stop`命令和容器名称来停止它：`nostalgic_morse`。

	$ docker stop nostalgic_morse

	nostalgic_morse

现在可以使用`docker ps`命令来检查容器是否已经停止。

	$ docker ps -l

## 重新启动Web应用程序容器

Oops！在您刚刚停止了容器后，您接到一个电话说另一个开发人员需要容器重新运行起来。从这里您有两个选择：您可以创建一个新的容器或重新启动旧的容器。现在我们来看看如何启动您之前的容器备份。

	$ docker start nostalgic_morse

	nostalgic_morse

现在，再次快速运行`docker ps -l`，以查看正在运行的容器是否已恢复或浏览容器的URL以查看应用程序是否响应。

> **注意：**
>也可以使用`docker restart`命令先停止然后在启动容器。

## 删除Web应用程序容器

您的同事已通知您，他们已使用完容器了，不再需要它了。现在，您可以使用`docker rm`命令删除它。

    $ docker rm nostalgic_morse

    Error: Impossible to remove a running container, please stop it first or use -f
    2014/05/24 08:12:56 Error: failed to remove one or more containers


发生了什么？您实际上不能删除正在运行的容器。这可以防止您意外删除可能需要的正在运行的容器。您可以通过先停止容器再次尝试。

	$ docker stop nostalgic_morse

	nostalgic_morse

	$ docker rm nostalgic_morse

	nostalgic_morse

现在容器被停止和删除。

> **注意：**
>记住移除容器是最后一步！

# 下一步

到现在为止，您只使用了从Docker Hub下载的镜像。接下来，您可以了解如何构建和共享您自己的镜像。

转到[使用Docker镜像](dockerimages.md)。