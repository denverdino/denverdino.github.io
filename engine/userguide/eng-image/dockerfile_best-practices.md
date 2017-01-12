---
description: Hints, tips and guidelines for writing clean, reliable Dockerfiles
keywords: Examples, Usage, base image, docker, documentation, dockerfile, best practices, hub, official repo
redirect_from:
- /articles/dockerfile_best-practices/
- /engine/articles/dockerfile_best-practices/
- /docker-cloud/getting-started/intermediate/optimize-dockerfiles/
- /docker-cloud/tutorials/optimize-dockerfiles/
title: Best practices for writing Dockerfiles
---

Docker 可以通过读取`Dockerfile`文件的指令来自动构建一个镜像，Dockerfile文件是一个文本文件，
包含构建需要的指令，是构建一个镜像的必备条件。`Dockerfile`遵守一定的格式，使用一个特定的指令集。你可以通过[Dockerfile 手册](../../reference/builder.md)来学习其基本知识，如果你是第一次写`Dockerfile`，你应该从基本知识开始。

这篇文档描述了由Docker公司和Docker社区推荐的编写`Dockerfile`的最佳实践，用来编写使用简单，有效的`Dockerfile`我们强烈建议你遵循这些建议(事实上，如果你创建了一个官方的镜像，你*必须*遵守这些实践)。

你可以在[buildpack-deps `Dockerfile`](https://github.com/docker-library/buildpack-deps/blob/master/jessie/Dockerfile)
中看到许多这样的实践和建议。

> 注意：对于本文中提到的任何Dockerfile指令的解释，请参考[Dockerfile 手册](../../reference/builder.md)

## 常规指南和建议

### 容器应该是短暂的

由`Dockerfile`定义的镜像创建出来的容器应该尽可能是短暂的。“短暂”意味着这个容器能被停止，销毁，
新的容器能被最小化的配置所启动起来，并部署到相应的位置。

### 使用.dockerignore文件

在大部分场景下，最好是将每一个Dockerfile文件放在一个空的目录下面。然后在该目录下面只添加构建Dockerfile
需要的文件。为了增加构建的性能，你可以通过在该目录中添加一个.dockerignore文件来排除文件或者目录。这个.dockerignore文件
支持的排除模式跟.gitignore文件类似。关于创建该文件的信息，请参考[.dockerignore 文件](../../reference/builder.md#dockerignore-file)。

### 避免安装不必要的软件包

为了减少复杂度，依赖，文件大小和构建时间，你应该避免安装额外的或者不必要的软件包，仅仅是因为它们看起来应该安装。
例如你不需要在一个数据库镜像中安装文本编辑器。

### 每个容器只运行一个进程。

在绝大部分场景下，你应该在容器中只运行一个进程。将应用解耦成多个容器，使之容易水平扩展和重用。
如果一个服务依赖于另外一个服务，利用[容器 linking](../../userguide/networking/default_network/dockerlinks.md)来关联。

### 最小化层的数量

你需要在`Dockerfile`的可读性(即长期的可维护性)和最小化层数量之间取得平衡。对于你使用的镜像层的
数量，请策略而谨慎的对待。

### 对多行的参数进行排序

在可能的时候，通过对多行的参数进行按照字母顺序的排序来使得后续的修改变得容易。这将帮助你重复的安装软件包和使得
软件包列表更加容易更新。这也使得PRs更加容易阅读和审阅。在每一个反斜杠(`\`)前面加上空格也会增加可读性。

这是一个来自[`buildpack-deps`镜像](https://github.com/docker-library/buildpack-deps)的例子：

    RUN apt-get update && apt-get install -y \
      bzr \
      cvs \
      git \
      mercurial \
      subversion

### 利用构建时的缓存

在构建的过程中，Docker会安装`Dockerfile`中指令列出的顺序逐步执行。检查每个要执行的指令前，Docker
会查找缓存中可以重复利用的已经存在的镜像，而不是重新去构建一个新的镜像。如果你不需要使用缓存，你可以
给`docker build`命令加上`--no-cache=true`的选项

然而，如果你让Docker在构建时使用缓存，理解什么时候Docker会去查找缓存中一个匹配的镜像是非常重要的
。使用缓存的基本规则罗列如下：


* 从已经在缓存中的基础镜像开始，下一条指令会与该基础镜像派生的所有子镜像进行对比，判断是否其中一个子镜像
使用了相同的指令，如果不是，则缓存是无效的。

* 在大多数情况下，只需要比较子镜像的指令和当前指令就足够了。但是某些指令需要更多的检查和解释。

* 对于`ADD`和`COPY`指令，会检查镜像中文件的内容，计算其校验和。校验和不会考虑文件的最后修改和最后访问的时间。在缓存查找过程中，会与缓存镜像中的校验和进行比对。如果文件内容发生了变化，例如文件的内容或者元信息，那么缓存是无效的

* 除了`ADD`和`COPY`命令，缓存检查不会查看容器中的文件来决定是否命中缓存。例如处理命令`RUN apt-get -y update`时
容器中被更新的文件不会被检查来确定是否存在缓存命中。在这个例子中，只有命令字符串本身会被比较。

一旦缓存无效，`Dockerfile`中所有接下来的指令都会生成新的镜像，缓存不会再被使用。

## Dockerfile 命令

下面你会发现一些关于如果使用最好的方式来编写`Dockerfile`中各种各样指令的建议。

### FROM

[Dockerfile 手册中的From命令](../../reference/builder.md#from)

只要有可能，使用官方仓库作为你镜像的基础。我们推荐[Debian镜像](https://hub.docker.com/_/debian/)，
该镜像被严格控制，并且保持最小大小（目前低于150 mb），同时还是一个完整的发行版。

### LABEL

[理解对象的标签](../labels-custom-metadata.md)

你可以给你的镜像添加标签来帮助根据应用组织镜像，记录许可信息，辅助自动构建，或者用于其他场景。对于每一个标签
增加以`LABEL`开头的一行以及加上一个或者多个键值对。下面的例子展示了不同的可以被接受的格式
解析的评论内联在示例中。

>**注意**：如果你的字符串中包含空格，它必须被"""引用起来或者对空格进行转义。如果你的字符串中包含内部双引号
(`"`)，对该双引号也要进行转义

```dockerfile
# 设置一个或者多个独立的标签
# Set one or more individual labels
LABEL com.example.version="0.0.1-beta"
LABEL vendor="ACME Incorporated"
LABEL com.example.release-date="2015-02-12"
LABEL com.example.version.is-production=""

# 在一行中设置多个标签
# Set multiple labels on one line
LABEL com.example.version="0.0.1-beta" com.example.release-date="2015-02-12"

# 一次性设置多个标签，使用行链接符号来将长的行分开
# Set multiple labels at once, using line-continuation characters to break long lines
LABEL vendor=ACME\ Incorporated \
      com.example.is-beta= \
      com.example.is-production="" \
      com.example.version="0.0.1-beta" \
      com.example.release-date="2015-02-12"
```

查看[理解对象的标签](../labels-custom-metadata.md)来查看关于可接受的标签键和值的指南。关于
查询标签的信息，参考位于[管理对象上的标签](../labels-custom-metadata.md#managing-labels-on-objects)的
关于过滤的内容项。

### RUN

[Dockerfile 手册的RUN指令](../../reference/builder.md#run)

和通常一样，为了让你的`Dockerfile`更具可读性，理解下性，以及维护性，将长的或者复杂的`RUN`指令
分成多行，使用反斜杠分隔。

#### apt-get

或许RUN指令最常见的用例是`apt-get`的应用。`RUN apt-get`命令，由于它安装软件包，有几个陷阱需要
注意。

你应该避免使用`RUN apt-get upgrade` 或者 `dist-upgrade`，因为许多在基础镜像中的必备软件包
不能在一个没有特权的容器中升级。如果在基础镜像中的软件包过期了，你应该联系这个基础镜像的维护者。
如果你知道有一个特定的软件包，例如`foo`需要更新，使用命令`apt-get install -y foo`来自动更新。

总是记住将`RUN apt-get update` 和 `apt-get install`放在同一个`RUN`指令中，例如：

        RUN apt-get update && apt-get install -y \
            package-bar \
            package-baz \
            package-foo


在`RUN`指令中独立地使用`apt-get update`会导致缓存问题以及接下来的`apt-get install`命令的失败。例如，你有一个Dockerfile：

        FROM ubuntu:14.04
        RUN apt-get update
        RUN apt-get install -y curl

构建完该镜像之后，所有的层都位于Docker缓存中。假设你随后通过增加额外的软件包修改了`apt-get install`：

        FROM ubuntu:14.04
        RUN apt-get update
        RUN apt-get install -y curl nginx

Docker 认为原始的命令和修改的命令是一样的，然后重用了之前构建的缓存。因此命令`apt-get update`由于使用了
缓存的版本而没有被执行。由于`apt-get update`没有被执行，你的构建可能会获得一个旧版本的`curl`和`nginx`
软件包。

使用 `RUN apt-get update && apt-get install -y`保证你的Dockerfile安装最新的软件包版本
而不需要进一步的编码或者手工的干预。这一技术称作"缓存无效化"。你也可以通过指定特定的软件包版本
来实现缓存无效化。这称作版本固定，例如：

        RUN apt-get update && apt-get install -y \
            package-bar \
            package-baz \
            package-foo=1.3.*

版本固定强制构建获取特定的软件包版本而忽略缓存的因素。这一技术也能减少软件包不可预知的更改导致的失败。

下面是一个组织好的`RUN`命令来展示所有`apt-get`的推荐用法。

    RUN apt-get update && apt-get install -y \
        aufs-tools \
        automake \
        build-essential \
        curl \
        dpkg-sig \
        libcap-dev \
        libsqlite3-dev \
        mercurial \
        reprepro \
        ruby1.9.1 \
        ruby1.9.1-dev \
        s3cmd=1.1.* \
     && rm -rf /var/lib/apt/lists/*

`s3cmd`指令指定了版本号`1.1.0*`。如果这个镜像之前使用了一个旧的版本，指定新的版本会导致`apt-get update`缓存无效化，
保证新版本的安装。在每一行中列出软件包也阻止软件包重复的错误。

另外，清理apt缓存和删除`/var/lib/apt/lists`帮助减小镜像大小。由于`RUN`命令以`apt-get update`开头，软件包的缓存始终会在`apt-get install`之前被刷新。

> **注意**：官方的Debian和Ubuntu镜像[自动运行`apt-get clean`](https://github.com/docker/docker/blob/03e2923e42446dbb830c654d0eec323a0b4ef02a/contrib/mkimage/debootstrap#L82-L105)
> 因此显式的执行是不需要的。

### CMD

[Dockerfile手册中的CMD指令](../../reference/builder.md#cmd)

`CMD`指令，携带着应用的参数，被用来执行镜像容器中的软件。`CMD`指令应该始终以CMD [“executable”, “param1”, “param2”…]`
的形式被使用。因此，如果镜像是被用来作为一个服务，例如Apache或者Rails服务，你将执行类似`CMD ["apache2","-DFOREGROUND"]`
的命令。如果是基于服务的镜像，推荐大家使用这种形式的指令。

在大部分其他的场景，应该给`CMD`指定一个交互式的shell，例如一个bash，python或者perl。举几个例子
`CMD ["perl", "-de0"]`， `CMD ["python"]`， 或者`CMD [“php”, “-a”]`。使用这种形式，意味着当你
执行像`docker run -it python`这样的指令时，你能进入一个立马能用的shell环境。`CMD`很罕见地以`CMD [“param”, “param”]`
这样的形式与[`ENTRYPOINT`](../../reference/builder.md#entrypoint)结合使用，除非你和你希望的
用户已经对`ENTRYPOINT`的用法非常熟悉了。

### EXPOSE

[Dockerfile手册中的EXPOSE指令](../../reference/builder.md#expose)

`EXPOSE`指令指明了容器监听链接时暴露的端口。因此，你应该让你的应用使用常见的，传统的端口。例如
一个包含Apache网页服务器的镜像会使用`EXPOSE 80`，当一个镜像包含MongoDB时，应该使用`EXPOSE 27017`
端口，以此类推。

对于外部的访问，你的镜像用户可以在执行`docker run`时携带一个参数来指明映射容器端口到相应的主机端口。
对于容器linking，Docker会提供环境变量给容器linking的接收者，(例如`MYSQL_PORT_3306_TCP`)。

### ENV


[Dockerfile 手册中的ENV命令](../../reference/builder.md#env)

为了让软件更容易执行，你可以使用`ENV`来更新`PATH`环境变量。例如 `ENV PATH /usr/local/nginx/bin:$PATH`
可以保证命令`CMD ["nging"]`能工作

`ENV`指令对于提供你需要容器化的应用的环境变量来说是有用的，例如Postgres的`PGDATA`环境变量。


最后，`ENV`也能用来设置常用的版本号，这样版本的变化变得容易维护了，如下所示：

    ENV PG_MAJOR 9.3
    ENV PG_VERSION 9.3.4
    RUN curl -SL http://example.com/postgres-$PG_VERSION.tar.xz | tar -xJC /usr/src/postgress && …
    ENV PATH /usr/local/postgres-$PG_MAJOR/bin:$PATH

类似于在程序中有常量而不是硬编码的值，这种方法让你能够通过修改一个单一的`ENV`指令来
自动对你容器的中版本进行升级。

### ADD or COPY

[Dockerfile手册中的ADD指令](../../reference/builder.md#add)<br/>
[Dockerfile手册中的COPY指令](../../reference/builder.md#copy)

通常来说，尽管`ADD`和`COPY`指令的功能类似的，但是推荐使用`COPY`指令。因为它与`ADD`指令相比
更加透明。`COPY`仅支持将本地文件拷贝到容器中的基本命令，而`ADD`命令有一些特性（像本地的tar包解压缩
和远程URL的支持）是不明显的。因此，对`ADD`来说最好的使用场景是本地tar包的解压缩到镜像中，例如
`ADD rootfs.tar.xz /`。

如果你有多个`Dockerfile`步骤，使用了上下文中不同的文件，对每个文件单独使用`COPY`指令而不是一次性拷贝
所有的文件。这会保证每一步骤的缓存只有在当前步骤的文件发生修改时，才会失效。

例如

    COPY requirements.txt /tmp/
    RUN pip install --requirement /tmp/requirements.txt
    COPY . /tmp/

对`RUN`步骤来说，与将指令`COPY . /tmp/`放在`RUN`指令之前相比，会导致更少的缓存失效

由于镜像大小很重要，不推荐使用`ADD`指令来从远程URL地址获取软件包；你应该使用`curl`或者`wget`。
这样你就可以在解压缩之后，删除你不再需要的文件以及你不需要在你的镜像中增加另外一层。例如你应该避免这样做：

    ADD http://example.com/big.tar.xz /usr/src/things/
    RUN tar -xJf /usr/src/things/big.tar.xz -C /usr/src/things
    RUN make -C /usr/src/things all

而是应该这样做：

    RUN mkdir -p /usr/src/things \
        && curl -SL http://example.com/big.tar.xz \
        | tar -xJC /usr/src/things \
        && make -C /usr/src/things all

对于其他项目（文件，目录），如果不需要`ADD`命令的自动解压缩功能，你应该始终使用`COPY`命令。

### ENTRYPOINT

[Dockerfile手册中的ENTRYPOINT指令](../../reference/builder.md#entrypoint)

对于`ENTRYPOINT`来说，最佳的目的是设置镜像的主要执行命令，运行这个镜像看起来就像是这个命令一
样(然后使用命令`CMD`来添加默认的参数)

让我们以一个命令行工具`s3cmd`镜像的例子开始：

    ENTRYPOINT ["s3cmd"]
    CMD ["--help"]

现在这个镜像可以像如下这样执行来显示这个命令的帮助内容：

    $ docker run s3cmd

或者使用一个正确是参数来执行一个命令：

    $ docker run s3cmd ls s3://mybucket

这是有用的，由于镜像的名称可以被当作是对可执行文件的引用，例如上面提到的例子。

`ENTRYPOINT`指令也能和帮助脚本结合进行使用，允许该命令以一种与上述命令相似的方式起作用，即使启动
这个工具需要多个步骤。

举例而言，[Postgres官方镜像](https://hub.docker.com/_/postgres/)使用下面的脚本来作为其
`ENTRYPOINT`：

```bash
#!/bin/bash
set -e

if [ "$1" = 'postgres' ]; then
    chown -R postgres "$PGDATA"

    if [ -z "$(ls -A "$PGDATA")" ]; then
        gosu postgres initdb
    fi

    exec gosu postgres "$@"
fi

exec "$@"
```

> **注意**：
> 这个脚本使用了[Bash中的`exec`命令](http://wiki.bash-hackers.org/commands/builtin/exec)
> 因此最后运行的应用程序变成了容器的1号进程。这允许应用程序接受任何传递给容器的Unix信号。
> 访问[`ENTRYPOINT`](../../reference/builder.md#entrypoint)查看更多的细节。


帮助脚本被拷贝进容器中，并通过`ENTRYPOINT`命令在容器启动时执行：

    COPY ./docker-entrypoint.sh /
    ENTRYPOINT ["/docker-entrypoint.sh"]

这个脚本允许用户通过几钟方式跟Postgrs进行交互。

它能简单地启动Postgres：

    $ docker run postgres

或者，它能被用来执行Postgres并将参数传递给服务器：

    $ docker run postgres postgres --help

最后，它还能被用来执行一个完全不同的工具，例如Bash：

    $ docker run --rm -it postgres bash

### VOLUME

[Dockerfile手册中的VOLUME命令](../../reference/builder.md#volume)

`VOLUME`指令被用来暴露任何数据库存储的区域，配置存储，或者由你的docker容器创建的文件或文件夹。强烈
建议你对镜像中可变的或者对用户提供服务的部分使用`VOLUME`指令。

### USER

[Dockerfile手册中的USER指令](../../reference/builder.md#user)

如果一个服务可以在没有特权的情况下运行，使用`USER`指令来修改为一个非root用户。在使用该指令前
通过`RUN groupadd -r postgres && useradd -r -g postgres postgres`来创建用户和组。

> **注意：** 镜像中的用户和组会被分配一个不确定的
> UID或者GID，由于不管镜像是否被重新构建，“下一个”UID或者GID都会被分配。
> 因此，如果UID或者GID是关键点，你应该显式分配它们。

由于`sudo`有不可预测的TTY或者信号转发行为，这可能会导致更多的问题，因此你应该避免安装或者
使用该命令。如果你必须使用跟`sudo`功能类似的命令(例如，将守护进程初始化为root用户，但是执行时变为非root用户)，
你可以考虑使用["gosu"](https://github.com/tianon/gosu)。

最后，为了避免减少层的次数和复杂度，避免来回切换使用`USER`命令。

### WORKDIR

[Dockerfile手册中的WORKDIR指令](../../reference/builder.md#workdir)

为了追求清晰和可读性，你应该总是在`WORKDIR`中使用绝对的路径。同时，你应该使用`WORKDIR`
而不是繁复的类似`RUN cd … && do-something`这样难以阅读，难以错误定位，难以维护的命令。

### ONBUILD

[Dockerfile手册中的ONBUILD指令](../../reference/builder.md#onbuild)

`ONBUILD`命令是在当前`Dockerfile`构建执行完毕之后再执行的。
`ONBUILD`是在任何使用`FROM`命令从当前镜像派生的子镜像中执行的。将`ONBUILD`看作是父`Dockerfile`
交给子`Dockerfile`执行的命令。

一次Docker构建，会在执行任何子`Dockerfile`命令之前执行`ONBUILD`命令。

`ONBUILD`在`FROM`一个给定的镜像下构建新的镜像是有用的。例如，你会在一个语言栈中使用`ONBUILD`
命令，这个镜像是用来构建任意以该语言编写的软件的`Dockerfile`而准备的，例如，你可以参考
[Ruby的`ONBUILD`变体](https://github.com/docker-library/ruby/blob/master/2.1/onbuild/Dockerfile)

从`ONBUILD`构建的镜像应该拿到一个有区分的tag，例如`ruby:1.9-onbuild` 或者 `ruby:2.0-onbuild`。


当你需要在`ONBUILD`中放入`ADD`或者`COPY`指令时，请务必谨慎。由于如果新的构建的上下文没有被添加的
资源，则"onbuild"的镜像会灾难性地失败。像上面推荐的那样，增加一个易于区分的tag，将通过允许`Dockerfile`的作者
作出选择来减少这种情况。

## 官方仓库的例子


这些官方的镜像仓库有一些作为样例的`Dockerfile`：

* [Go](https://hub.docker.com/_/golang/)
* [Perl](https://hub.docker.com/_/perl/)
* [Hy](https://hub.docker.com/_/hylang/)
* [Rails](https://hub.docker.com/_/rails)

## 额外的资源

* [Dockerfile 手册](../../reference/builder.md)
* [深入基础镜像](baseimages.md)
* [深入自动化镜像构建](/docker-hub/builds/)
* [创建官方镜像仓库指南](/docker-hub/official_repos/)
