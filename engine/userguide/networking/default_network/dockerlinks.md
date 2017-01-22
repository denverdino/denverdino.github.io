---
description: Learn how to connect Docker containers together.
keywords: Examples, Usage, user guide, links, linking, docker, documentation, examples, names, name, container naming, port, map, network port, network
redirect_from:
- /userguide/dockerlinks/
title: Legacy container links
---

本节中的信息说明了在安装Docker时自动创建的Docker默认网桥网络中的旧容器链接。

在[Docker网络功能](../目录.md)之前，您可以使用Docker链接功能来允许容器发现彼此，并将一个容器的信息安全地传输到另一个容器。 通过引入Docker网络功能，您仍然可以创建链接，但是它们在默认网桥网络和[用户定义的网络](../work-with-networks.md#linking-containers-in-user-defined-networks)之间的行为不同

本节先简要讨论通过网络端口连接，然后详细理解在默认`bridge`网络中的容器链接情况。

> **警告**：`--link`标志是Docker遗弃的旧特性。它可能最终会被删除。除非您绝对需要继续使用它，我们建议您使用用户定义的网络，以方便两个容器之间的通信而不是使用`--link`。使用`--link`可以在容器之间共享环境变量，而使用用户自定义的网络不支持这个功能。然而，您可以使用其他机制（如卷）共享环境变量。

##使用网络端口映射连接

在[运行一个简单的应用程序](../../../tutorials/usingdocker.md)，您创建了一个
容器运行Python Flask应用程序：

    $ docker run -d -P training/webapp python app.py

> **注意：** 容器具有内部网络和IP地址（正如我们在[运行一个简单的应用程序](../../../ tutorials / usingdocker.md)中使用`docker inspect`命令查看容器IP地址时候看到的一样。Docker可以有多种网络配置。 您可以看到更多关于Docker网络的信息[这里](../ index.md)。

创建该容器时，使用`-P`标志自动将其中的任何网络端口映射到Docker主机上的*临时端口范围*内的随机高端口。 接下来，当docker ps运行时，您看到容器中的端口5000绑定到主机上的端口49155。

    $ docker ps nostalgic_morse

    CONTAINER ID  IMAGE                   COMMAND       CREATED        STATUS        PORTS                    NAMES
    bc533791f3f5  training/webapp:latest  python app.py 5 seconds ago  Up 2 seconds  0.0.0.0:49155->5000/tcp  nostalgic_morse

您也看到如何使用`-p`标志将容器端口绑定到特定端口。这里主机的端口`80`映射到容器的端口`5000`：

    $ docker run -d -p 80:5000 training/webapp python app.py

您知道为什么这不是一个好主意，因为它限制您只有一个特定端口上的容器。
相反，您可以指定绑定容器端口的主机端口范围不同于默认*临时端口范围*：

    $ docker run -d -p 8000-9000:5000 training/webapp python app.py

这将绑定容器5000到主机上8000和9000之间的随机可用端口。

还有一些其他方法可以配置`-p`标志。默认情况下，`-p`标志将指定的端口绑定到主机上的所有接口。 但是您也可以指定对特定接口的绑定，例如仅指向`localhost`。

    $ docker run -d -p 127.0.0.1:80:5000 training/webapp python app.py

这将绑定容器内的端口5000到`localhost`的端口80或主机上的`127.0.0.1`接口。

或者，要将容器的端口5000绑定到动态端口，但只能在`localhost`上，您可以使用：

    $ docker run -d -p 127.0.0.1::5000 training/webapp python app.py

您还可以通过添加尾部`/udp`来绑定UDP端口。 例如：

    $ docker run -d -p 127.0.0.1:80:5000/udp training/webapp python app.py

您还了解了有用的`docker port`快捷方式，其中显示了当前的端口绑定。 这也有助于显示特定的端口配置。 例如，如果您已将容器端口绑定到主机上的本地主机，则`docker port`输出将反映该情况。

    $ docker port nostalgic_morse 5000

    127.0.0.1:49155

> **注意**
>`-p`标志可以多次使用以配置多个端口。

## 使用链接系统连接
> **注意**: 本节涵盖默认`bridge`网络中的旧链接功能。有关用户定义网络中的链接的更多信息。请参考[用户定义网络中的链接容器](../ work-with-networks.md＃linking-containers-in-user-defined-networks)

网络端口映射不是Docker容器可以彼此连接的唯一方式。 Docker还有一个链接系统，允许您将多个容器链接在一起，并将连接信息从一个发送到另一个。 当容器被链接时，关于源容器的信息可以被发送到接收容器。 这允许接收者看到描述源容器的数据。

## 命名的重要性
为了建立链接，Docker依赖于您的容器的名字。 您已经看到，您创建的每个容器都有自动创建的名称; 在本指南中，您已经熟悉我们的老朋友`nostalgic_morse`在本指南。 您也可以自己命名容器。 这个命名提供了两个有用的功能：

1. 以便于记住它们的方式命名具有特定功能的容器很有用，例如把包含Web应用程序的容器命名为`web`。

2. 它为Docker提供了一个引用点，允许它引用其他容器，例如，您可以指定将容器`Web`链接到容器`db`。

您可以使用`--name`标志命名容器，例如：

    $ docker run -d -P --name web training/webapp python app.py

这将启动一个新容器，并使用`--name`标志来命名容器`web`。 您可以使用`docker ps`命令查看容器的名称。

    $ docker ps -l

    CONTAINER ID  IMAGE                  COMMAND        CREATED       STATUS       PORTS                    NAMES
    aed84ee21bde  training/webapp:latest python app.py  12 hours ago  Up 2 seconds 0.0.0.0:49154->5000/tcp  web

您也可以使用`docker inspect`返回容器的名称

> ** 注意 **
容器名称必须是唯一的。 这意味着您只能调用一个容器`web`。 如果要重新使用容器名称，必须先删除旧容器（使用`docker rm`），然后才能创建具有相同名称的新容器。 作为替代，可以在使用`docker run`命令的时候使用`--rm`标志。 这将在容器停止后立即删除它。

## 通过链接通信
链接允许容器发现彼此，并将一个容器的信息安全地传送到另一个容器。 设置链接时，将在源容器和接收容器之间创建一个管道。然后，接收者可以访问有关源的选择数据。 要创建链接，请使用`--link`标志。 首先，创建一个新的容器，这一次包含一个数据库。

    $ docker run -d --name db training/postgres

这将以`training/postgres`镜像创建一个名为`db`的新容器，其中包含一个PostgreSQL数据库。
现在，您需要删除先前创建的Web容器，以便您可以使用链接的容器替换它：

    $ docker run -d --name db training/postgres

现在，创建一个新的`web`容器并将其与您的`db`容器相链接。

    $ docker run -d -P --name web --link db:db training/webapp python app.py

这将链接新的`web`容器与您之前创建的`db`容器。该`--link`标志的形式为：

    --link <name or id>:alias

其中name是我们链接到的容器的名称，alias是链接名称的别名。 您会看到如何使用别名很快使用。 --link标志也采取以下形式：

	--link <name or id>

在这种情况下，别名将匹配名称。 您可以把前面的例子写成：

    $ docker run -d -P --name web --link db training/webapp python app.py

接下来，用`docker inspect`检查链接的容器：  

    {% raw %}
    $ docker inspect -f "{{ .HostConfig.Links }}" web

    [/db:/web/db]
    {% endraw %}

您可以看到链接`web/db`，表示`web`容器现在链接到`db`容器。 它允许`web`容器访问有关`db`容器的信息。

那么容器链接实际上做了什么？ 您已经了解到，链接允许源容器向接收容器提供有关自身的信息。 在我们的示例中，接收者`web`可以访问有关`db`的信息。 为此，Docker在容器之间创建一个不需要在容器外部暴露任何端口的安全隧道; 您会注意到当我们启动db容器时，我们没有使用`-P`或`-p`标志。 这是链接的一个很大的好处：我们不需要将源容器（这里是PostgreSQL数据库）暴露给网络。

Docker以两种方式将源容器的连接信息暴露给接收容器：
* 环境变量
* 更新`/etc/hosts`文件

## 环境变量
当您链接容器时，Docker会创建几个环境变量。Docker基于`--link`参数在目标容器中自动创建环境变量。 它还把源容器中所有环境变量暴露给目标容器。 这些变量包括：
* 源容器的Dockerfile中`ENV`命令所指定的
* 源容器启动时，`docker run`命令的`-e`，`--env`和`--env-file`选项所指定的

这些环境变量允许从目标容器内, 程序化的发现源容器的相关信息。

> **警告**：理解一个容器中的所有环境变量都可以通过Docker被它所链接到的容器获取到是非常重要的。如果敏感数据存储在其中，这可能具有严重的安全隐患。

Docker为`--link`参数中列出的每个目标容器设置一个`<alias> _NAME`环境变量。例如，如果一个称为`web`的新容器通过`--link db：webdb`链接到名为`db`的数据库容器，则Docker在`web`容器中创建一个`WEBDB_NAME = /web/webdb`变量。

Docker还为源容器公开的每个端口定义了一组环境变量。 每个变量具有以下形式的唯一前缀：

`<name>_PORT_<port>_<protocol>`

此前缀中的组成部分是：
* 在`--link`参数中指定的别名`<name>`（例如`webdb`）
* `<port>`暴露的数字
* `<protocol>`是TCP或UDP

Docker使用此前缀格式定义三个不同的环境变量：
* `prefix_ADDR`变量包含来自URL的IP地址，例如`WEBDB_PORT_5432_TCP_ADDR = 172.17.0.82`。
* `prefix_PORT`变量只包含来自URL的端口号，例如`WEBDB_PORT_5432_TCP_PORT = 5432`。
* `prefix_PROTO`变量只包含来自URL的协议，例如`WEBDB_PORT_5432_TCP_PROTO = tcp`。

如果容器暴露多个端口，则为每个端口定义一个环境变量集。这意味着，如果一个容器暴露4个端口，Docker创建12个环境变量，每个端口3个。

另外，Docker创建了一个名为`<alias> _PORT`的环境变量。此变量包含源容器的第一个公开端口的URL。 “第一”端口被定义为具有最低编号的暴露端口。例如，考虑`WEBDB_PORT=tcp//172.17.0.825432`变量。如果该端口用于tcp和udp，则指定tcp。

最后，Docker还将来自源容器的每个Docker生成的环境变量作为目标中的环境变量。对于每个变量，Docker在目标容器中创建一个`<alias> _ENV_ <name>`变量。变量的值设置为Docker启动源容器时使用的值。

返回到我们的数据库示例，您可以运行`env`命令来列出指定容器的环境变量。

```
    $ docker run --rm --name web2 --link db:db training/webapp env

    . . .
    DB_NAME=/web2/db
    DB_PORT=tcp://172.17.0.5:5432
    DB_PORT_5432_TCP=tcp://172.17.0.5:5432
    DB_PORT_5432_TCP_PROTO=tcp
    DB_PORT_5432_TCP_PORT=5432
    DB_PORT_5432_TCP_ADDR=172.17.0.5
    . . .
```

您可以看到Docker已经创建了一系列环境变量以及有关源数据库容器的有用信息。 每个变量前缀有`DB_`，它是从上面指定的别名填充的。 如果别名是`db1`，那么变量将以`DB1_`作为前缀。 您可以使用这些环境变量来配置应用程序以连接到db容器上的数据库。 连接将是安全的和私人的; 只有链接的`web`容器才能够与`db`容器通信。

### Docker环境变量的重要注意事项
与[`/etc/hosts`文件](#updating-the-etchosts-file)中的主机条目不同，如果重新启动源容器，是不会自动更新存储在环境变量中的IP地址的。我们建议使用`/etc/hosts`中的主机条目来解析链接容器的IP地址。

这些环境变量只对容器中的第一个进程设置。 一些守护进程（如`sshd`）会在生成shell进行连接时擦除它们。

### 更新`/etc/hosts`file

除了环境变量，Docker还会将源容器的主机条目添加到`/etc/hosts`文件中。这里是一个web容器的条目：

    $ docker run -t -i --rm --link db:webdb training/webapp /bin/bash

    root@aed84ee21bde:/opt/webapp# cat /etc/hosts

    172.17.0.7  aed84ee21bde
    . . .
    172.17.0.5  webdb 6e5cdeb2d300 db

您可以看到两个相关的主机条目。 第一个是`web`容器的条目，使用容器ID作为主机名。第二个条目使用链接别名以引用`db`容器的IP地址。 此外您提供的别名，链接的容器的名称(如果与提供给`--link`参数别名不同)，链接容器的主机名，也可以在`/etc/hosts`中为链接的容器的IP地址添加。 您现在可以通过以下任何方式`ping`那台主机:

    root@aed84ee21bde:/opt/webapp# apt-get install -yqq inetutils-ping

    root@aed84ee21bde:/opt/webapp# ping webdb

    PING webdb (172.17.0.5): 48 data bytes
    56 bytes from 172.17.0.5: icmp_seq=0 ttl=64 time=0.267 ms
    56 bytes from 172.17.0.5: icmp_seq=1 ttl=64 time=0.250 ms
    56 bytes from 172.17.0.5: icmp_seq=2 ttl=64 time=0.256 ms

> **注意**：在示例中，您将注意到，您必须安装ping，因为它最初未包含在容器中。

在这里，您使用`ping`命令使用其主机条目对`db`容器执行ping操作，该条目解析为`172.17.0.5`。您可以使用此主机条目配置应用程序以使用您的`db`容器。  

> **注意**：您可以将多个接收容器链接到单个源。例如，您可以将多个（不同命名的）Web容器附加到您的db容器。

如果重新启动源容器，链接的容器`/etc/hosts`文件将自动更新为源容器的新IP地址，从而允许链接的通信继续。

    $ docker restart db

    db

    $ docker run -t -i --rm --link db:db training/webapp /bin/bash

    root@aed84ee21bde:/opt/webapp# cat /etc/hosts

    172.17.0.7  aed84ee21bde
    . . .
    172.17.0.9  db

# 相关信息
