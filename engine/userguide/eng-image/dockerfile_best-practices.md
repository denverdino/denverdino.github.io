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
包含构建需要的指令，是构建一个镜像的必备条件。`Dockerfile`遵守一定的格式，使用一个特定的指令集。你
可以通过[Dockerfile 手册](../../reference/builder.md)来学习其基本知识，如果你是第一次写`Dockerfile`
，你应该从基本知识开始。
Docker can build images automatically by reading the instructions from a
`Dockerfile`, a text file that contains all the commands, in order, needed to
build a given image. `Dockerfile`s adhere to a specific format and use a
specific set of instructions. You can learn the basics on the
[Dockerfile Reference](../../reference/builder.md) page. If
you’re new to writing `Dockerfile`s, you should start there.

这篇文档描述了由Docker公司和Docker社区推荐的编写`Dockerfile`的最佳实践，用来创建使用简单，有效的`Dockerfile`
我们强烈建议你遵循这些建议(事实上，如果你创建了一个官方的镜像，你*必须*遵守这些实践)。
This document covers the best practices and methods recommended by Docker,
Inc. and the Docker community for creating easy-to-use, effective
`Dockerfile`s. We strongly suggest you follow these recommendations (in fact,
if you’re creating an Official Image, you *must* adhere to these practices).

你可以在[buildpack-deps `Dockerfile`](https://github.com/docker-library/buildpack-deps/blob/master/jessie/Dockerfile)
中看到许多这样的实践和建议。
You can see many of these practices and recommendations in action in the [buildpack-deps `Dockerfile`](https://github.com/docker-library/buildpack-deps/blob/master/jessie/Dockerfile).

> 注意：对于本文中提到的任何Dockerfile指令的解释，请参考[Dockerfile 手册](../../reference/builder.md)
> Note: for more detailed explanations of any of the Dockerfile commands
>mentioned here, visit the [Dockerfile Reference](../../reference/builder.md) page.

## 常规指南和建议
## General guidelines and recommendations

### 容器应该是短暂的
### Containers should be ephemeral

由`Dockerfile`定义的镜像创建出来的容器应该尽可能是短暂的。“短暂”意味着这个容器能被停止，销毁，
新的容器能被最小化的配置所启动起来，并部署到相应的位置。
The container produced by the image your `Dockerfile` defines should be as
ephemeral as possible. By “ephemeral,” we mean that it can be stopped and
destroyed and a new one built and put in place with an absolute minimum of
set-up and configuration.

### 使用.dockerignore文件
### Use a .dockerignore file

在大部分场景下，最好是将每一个Dockerfile文件放在一个空的目录下面。然后在该目录下面只添加构建Dockerfile
需要的文件。为了增加构建的性能，你可以通过在该目录中添加一个.dockerignore文件来排除文件或者目录。这个.dockerignore文件
支持的排除模式跟.gitignore文件类似。关于创建该文件的信息，请参考[.dockerignore 文件](../../reference/builder.md#dockerignore-file)。
In most cases, it's best to put each Dockerfile in an empty directory. Then,
add to that directory only the files needed for building the Dockerfile. To
increase the build's performance, you can exclude files and directories by
adding a `.dockerignore` file to that directory as well. This file supports
exclusion patterns similar to `.gitignore` files. For information on creating one,
see the [.dockerignore file](../../reference/builder.md#dockerignore-file).

### 避免安装不必要的软件包
### Avoid installing unnecessary packages

为了减少复杂度，依赖，文件大小和构建时间，你应该避免安装额外的或者不必要的软件包，仅仅是因为它们看起来应该安装。
例如你不需要在一个数据库镜像中安装文本编辑器。
In order to reduce complexity, dependencies, file sizes, and build times, you
should avoid installing extra or unnecessary packages just because they
might be “nice to have.” For example, you don’t need to include a text editor
in a database image.

### 每个容器只运行一个进程。
### Run only one process per container

在绝大部分场景下，你应该在容器中只运行一个进程。将应用解耦成多个容器，使之容易水平扩展和重用。
如果一个服务依赖于另外一个服务，利用[容器 linking](../../userguide/networking/default_network/dockerlinks.md)来关联。
In almost all cases, you should only run a single process in a single
container. Decoupling applications into multiple containers makes it much
easier to scale horizontally and reuse containers. If that service depends on
another service, make use of [container linking](../../userguide/networking/default_network/dockerlinks.md).

### 最小化层的数量
### Minimize the number of layers

你需要在`Dockerfile`的可读性(即长期的可维护性)和最小化层数量之间取得平衡。对于你使用的镜像层的
数量，请策略而谨慎的对待。
You need to find the balance between readability (and thus long-term
maintainability) of the `Dockerfile` and minimizing the number of layers it
uses. Be strategic and cautious about the number of layers you use.

### 对多行的参数进行排序
### Sort multi-line arguments

在可能的时候，通过对多行的参数进行按照字母顺序的排序来使得后续的修改变得容易。这将帮助你重复的安装软件包和使得
软件包列表更加容易更新。这也使得PRs更加容易阅读和审阅。在每一个反斜杠(`\`)前面加上空格也会增加可读性。
Whenever possible, ease later changes by sorting multi-line arguments
alphanumerically. This will help you avoid duplication of packages and make the
list much easier to update. This also makes PRs a lot easier to read and
review. Adding a space before a backslash (`\`) helps as well.

这是一个来自[`buildpack-deps`镜像](https://github.com/docker-library/buildpack-deps)的例子：
Here’s an example from the [`buildpack-deps` image](https://github.com/docker-library/buildpack-deps):

    RUN apt-get update && apt-get install -y \
      bzr \
      cvs \
      git \
      mercurial \
      subversion

### 利用构建时的缓存
### Build cache

在构建的过程中，Docker会安装`Dockerfile`中指令列出的顺序逐步执行。检查每个要执行的指令前，Docker
会查找缓存中可以重复利用的已经存在的镜像，而不是重新去构建一个新的镜像。如果你不需要使用缓存，你可以
给`docker build`命令加上`--no-cache=true`的选项
During the process of building an image Docker will step through the
instructions in your `Dockerfile` executing each in the order specified.
As each instruction is examined Docker will look for an existing image in its
cache that it can reuse, rather than creating a new (duplicate) image.
If you do not want to use the cache at all you can use the `--no-cache=true`
option on the `docker build` command.

然而，如果你让Docker在构建时使用缓存，理解什么时候Docker会去查找缓存中一个匹配的镜像是非常重要的
。使用缓存的基本规则罗列如下：
However, if you do let Docker use its cache then it is very important to
understand when it will, and will not, find a matching image. The basic rules
that Docker will follow are outlined below:


* 从已经在缓存中的基础镜像开始，下一条指令会与该基础镜像派生的所有子镜像进行对比，判断是否其中一个子镜像
使用了相同的指令，如果不是，则缓存是无效的。
* Starting with a base image that is already in the cache, the next
instruction is compared against all child images derived from that base
image to see if one of them was built using the exact same instruction. If
not, the cache is invalidated.

* 在大多数情况下，只需要比较子镜像的指令和当前指令就足够了。但是某些指令需要更多的检查和解释。
* In most cases simply comparing the instruction in the `Dockerfile` with one
of the child images is sufficient.  However, certain instructions require
a little more examination and explanation.

* 对于`ADD`和`COPY`指令，会检查镜像中文件的内容，计算其校验和。校验和不会考虑文件的最后修改和最后访问的时间。
在缓存查找过程中，会与缓存镜像中的校验和进行比对。如果文件内容发生了变化，例如文件的内容或者元信息，那么缓存是无效的
* For the `ADD` and `COPY` instructions, the contents of the file(s)
in the image are examined and a checksum is calculated for each file.
The last-modified and last-accessed times of the file(s) are not considered in
these checksums. During the cache lookup, the checksum is compared against the
checksum in the existing images. If anything has changed in the file(s), such
as the contents and metadata, then the cache is invalidated.

* 除了`ADD`和`COPY`命令，缓存检查不会查看容器中的文件来决定是否命中缓存。例如处理命令`RUN apt-get -y update`时
容器中被更新的文件不会被检查来确定是否存在缓存命中。在这个例子中，只有命令字符串本身会被比较。
* Aside from the `ADD` and `COPY` commands, cache checking will not look at the
files in the container to determine a cache match. For example, when processing
a `RUN apt-get -y update` command the files updated in the container
will not be examined to determine if a cache hit exists.  In that case just
the command string itself will be used to find a match.

一旦缓存无效，`Dockerfile`中所有接下来的指令都会生成新的镜像，缓存不会再被使用。
Once the cache is invalidated, all subsequent `Dockerfile` commands will
generate new images and the cache will not be used.

## Dockerfile 命令
## The Dockerfile instructions

下面你会发现一些关于如果使用最好的方式来编写`Dockerfile`中各种各样指令的建议。
Below you'll find recommendations for the best way to write the
various instructions available for use in a `Dockerfile`.

### FROM

[Dockerfile 手册中的From命令](../../reference/builder.md#from)
[Dockerfile reference for the FROM instruction](../../reference/builder.md#from)

只要有可能，使用官方仓库作为你镜像的基础。我们推荐[Debian镜像](https://hub.docker.com/_/debian/)，
该镜像被严格控制，并且保持最小大小（目前低于150 mb），同时还是一个完整的发行版。
Whenever possible, use current Official Repositories as the basis for your
image. We recommend the [Debian image](https://hub.docker.com/_/debian/)
since it’s very tightly controlled and kept minimal (currently under 150 mb),
while still being a full distribution.

### LABEL

[理解对象的标签](../labels-custom-metadata.md)
[Understanding object labels](../labels-custom-metadata.md)

你可以给你的镜像添加标签来帮助根据应用组织镜像，记录许可信息，辅助自动构建，或者用于其他场景。对于每一个标签
增加以`LABEL`开头的一行以及加上一个或者多个键值对。下面的例子展示了不同的可以被接受的格式
解析的评论内联在示例中。
You can add labels to your image to help organize images by project, record
licensing information, to aid in automation, or for other reasons. For each
label, add a line beginning with `LABEL` and with one or more key-value pairs.
The following examples show the different acceptable formats. Explanatory comments
are included inline.

>**注意**：如果你的字符串中包含空格，它必须被"""引用起来或者对空格进行转义。如果你的字符串中包含内部双引号
(`"`)，对该双引号也要进行转义
>**Note**: If your string contains spaces, it must be quoted **or** the spaces
must be escaped. If your string contains inner quote characters (`"`), escape
them as well.

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
See [Understanding object labels](../labels-custom-metadata.md) for
guidelines about acceptable label keys and values. For information about
querying labels, refer to the items related to filtering in
[Managing labels on objects](../labels-custom-metadata.md#managing-labels-on-objects).

### RUN

[Dockerfile 手册的RUN指令](../../reference/builder.md#run)
[Dockerfile reference for the RUN instruction](../../reference/builder.md#run)

和通常一样，为了让你的`Dockerfile`更具可读性，理解下性，以及维护性，将长的或者复杂的`RUN`指令
分成多行，使用反斜杠分隔。
As always, to make your `Dockerfile` more readable, understandable, and
maintainable, split long or complex `RUN` statements on multiple lines separated
with backslashes.

#### apt-get

或许RUN指令最常见的用例是`apt-get`的应用。`RUN apt-get`命令，由于它安装软件包，有几个陷阱需要
注意。
Probably the most common use-case for `RUN` is an application of `apt-get`. The
`RUN apt-get` command, because it installs packages, has several gotchas to look
out for.

你应该避免使用`RUN apt-get upgrade` 或者 `dist-upgrade`，因为许多在基础镜像中的必备软件包
不能在一个没有特权的容器中升级。如果在基础镜像中的软件包过期了，你应该联系这个基础镜像的维护者。
如果你知道有一个特定的软件包，例如`foo`需要更新，使用命令`apt-get install -y foo`来自动更新。
You should avoid `RUN apt-get upgrade` or `dist-upgrade`, as many of the
“essential” packages from the base images won't upgrade inside an unprivileged
container. If a package contained in the base image is out-of-date, you should
contact its maintainers.
If you know there’s a particular package, `foo`, that needs to be updated, use
`apt-get install -y foo` to update automatically.

总是记住将`RUN apt-get update` 和 `apt-get install`放在同一个`RUN`指令中，例如：
Always combine  `RUN apt-get update` with `apt-get install` in the same `RUN`
statement, for example:

        RUN apt-get update && apt-get install -y \
            package-bar \
            package-baz \
            package-foo


在`RUN`指令中独立地使用`apt-get update`会导致缓存问题以及接下来的`apt-get install`命令的失败。
Using `apt-get update` alone in a `RUN` statement causes caching issues and
subsequent `apt-get install` instructions fail.
For example, say you have a Dockerfile:
例如，你有一个Dockerfile：

        FROM ubuntu:14.04
        RUN apt-get update
        RUN apt-get install -y curl

构建完该镜像之后，所有的层都位于Docker缓存中。假设你随后通过增加额外的软件包修改了`apt-get install`：
After building the image, all layers are in the Docker cache. Suppose you later
modify `apt-get install` by adding extra package:

        FROM ubuntu:14.04
        RUN apt-get update
        RUN apt-get install -y curl nginx

Docker 认为原始的命令和修改的命令是一样的，然后重用了之前构建的缓存。因此命令`apt-get update`由于使用了
缓存的版本而没有被执行。由于`apt-get update`没有被执行，你的构建可能会获得一个旧版本的`curl`和`nginx`
软件包。
Docker sees the initial and modified instructions as identical and reuses the
cache from previous steps. As a result the `apt-get update` is *NOT* executed
because the build uses the cached version. Because the `apt-get update` is not
run, your build can potentially get an outdated version of the `curl` and `nginx`
packages.

使用 `RUN apt-get update && apt-get install -y`保证你的Dockerfile安装最新的软件包版本
而不需要进一步的编码或者手工的干预。这一技术称作"缓存无效化"。你也可以通过指定特定的软件包版本
来实现缓存无效化。这称作版本固定，例如：
Using  `RUN apt-get update && apt-get install -y` ensures your Dockerfile
installs the latest package versions with no further coding or manual
intervention. This technique is known as "cache busting". You can also achieve
cache-busting by specifying a package version. This is known as version pinning,
for example:

        RUN apt-get update && apt-get install -y \
            package-bar \
            package-baz \
            package-foo=1.3.*

版本固定强制构建获取特定的软件包版本而忽略缓存的因素。这一技术也能减少软件包不可预知的更改导致的失败。
Version pinning forces the build to retrieve a particular version regardless of
what’s in the cache. This technique can also reduce failures due to unanticipated changes
in required packages.

下面是一个组织好的`RUN`命令来展示所有`apt-get`的推荐用法。
Below is a well-formed `RUN` instruction that demonstrates all the `apt-get`
recommendations.

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
The `s3cmd` instructions specifies a version `1.1.0*`. If the image previously
used an older version, specifying the new one causes a cache bust of `apt-get
update` and ensure the installation of the new version. Listing packages on
each line can also prevent mistakes in package duplication.

另外，清理apt缓存和删除`/var/lib/apt/lists`帮助减小镜像大小。由于`RUN`命令以`apt-get update`开头，软件包
的缓存始终会在`apt-get install`之前被刷新。
In addition, cleaning up the apt cache and removing `/var/lib/apt/lists` helps
keep the image size down. Since the `RUN` statement starts with
`apt-get update`, the package cache will always be refreshed prior to
`apt-get install`.

> **注意**：官方的Debian和Ubuntu镜像[自动运行`apt-get clean`](https://github.com/docker/docker/blob/03e2923e42446dbb830c654d0eec323a0b4ef02a/contrib/mkimage/debootstrap#L82-L105)
> 因此显式的执行是不需要的。
> **Note**: The official Debian and Ubuntu images [automatically run `apt-get clean`](https://github.com/docker/docker/blob/03e2923e42446dbb830c654d0eec323a0b4ef02a/contrib/mkimage/debootstrap#L82-L105),
> so explicit invocation is not required.

### CMD

[Dockerfile手册中的CMD指令](../../reference/builder.md#cmd)
[Dockerfile reference for the CMD instruction](../../reference/builder.md#cmd)

`CMD`指令，携带着应用的参数，被用来执行镜像容器中的软件。`CMD`指令应该始终以CMD [“executable”, “param1”, “param2”…]`
的形式被使用。因此，如果镜像是被用来作为一个服务，例如Apache或者Rails服务，你将执行类似`CMD ["apache2","-DFOREGROUND"]`
的命令。如果是基于服务的镜像，推荐大家使用这种形式的指令。
The `CMD` instruction should be used to run the software contained by your
image, along with any arguments. `CMD` should almost always be used in the
form of `CMD [“executable”, “param1”, “param2”…]`. Thus, if the image is for a
service, such as Apache and Rails, you would run something like
`CMD ["apache2","-DFOREGROUND"]`. Indeed, this form of the instruction is
recommended for any service-based image.

在大部分其他的场景，应该给`CMD`指定一个交互式的shell，例如一个bash，python或者perl。举几个例子
`CMD ["perl", "-de0"]`， `CMD ["python"]`， 或者`CMD [“php”, “-a”]`。使用这种形式，意味着当你
执行像`docker run -it python`这样的指令时，你能进入一个立马能用的shell环境。`CMD`很罕见地以`CMD [“param”, “param”]`
这样的形式与[`ENTRYPOINT`](../../reference/builder.md#entrypoint)结合使用，除非你和你希望的
用户已经对`ENTRYPOINT`的用法非常熟悉了。
In most other cases, `CMD` should be given an interactive shell, such as bash, python
and perl. For example, `CMD ["perl", "-de0"]`, `CMD ["python"]`, or
`CMD [“php”, “-a”]`. Using this form means that when you execute something like
`docker run -it python`, you’ll get dropped into a usable shell, ready to go.
`CMD` should rarely be used in the manner of `CMD [“param”, “param”]` in
conjunction with [`ENTRYPOINT`](../../reference/builder.md#entrypoint), unless
you and your expected users are already quite familiar with how `ENTRYPOINT`
works.

### EXPOSE

[Dockerfile手册中的EXPOSE指令](../../reference/builder.md#expose)
[Dockerfile reference for the EXPOSE instruction](../../reference/builder.md#expose)

`EXPOSE`指令指明了容器监听链接时暴露的端口。因此，你应该让你的应用使用常见的，传统的端口。例如
一个包含Apache网页服务器的镜像会使用`EXPOSE 80`，当一个镜像包含MongoDB时，应该使用`EXPOSE 27017`
端口，以此类推。
The `EXPOSE` instruction indicates the ports on which a container will listen
for connections. Consequently, you should use the common, traditional port for
your application. For example, an image containing the Apache web server would
use `EXPOSE 80`, while an image containing MongoDB would use `EXPOSE 27017` and
so on.

对于外部的访问，你的镜像用户可以在执行`docker run`时携带一个参数来指明映射容器端口到相应的主机端口。
对于容器linking，Docker会提供环境变量给容器linking的接收者，(例如`MYSQL_PORT_3306_TCP`)。
For external access, your users can execute `docker run` with a flag indicating
how to map the specified port to the port of their choice.
For container linking, Docker provides environment variables for the path from
the recipient container back to the source (ie, `MYSQL_PORT_3306_TCP`).

### ENV


[Dockerfile 手册中的ENV命令](../../reference/builder.md#env)
[Dockerfile reference for the ENV instruction](../../reference/builder.md#env)

为了让软件更容易执行，你可以使用`ENV`来更新`PATH`环境变量。例如 `ENV PATH /usr/local/nginx/bin:$PATH`
可以保证命令`CMD ["nging"]`能工作
In order to make new software easier to run, you can use `ENV` to update the
`PATH` environment variable for the software your container installs. For
example, `ENV PATH /usr/local/nginx/bin:$PATH` will ensure that `CMD [“nginx”]`
just works.

`ENV`指令对于提供你需要容器化的应用的环境变量来说是有用的，例如Postgres的`PGDATA`环境变量。
The `ENV` instruction is also useful for providing required environment
variables specific to services you wish to containerize, such as Postgres’s
`PGDATA`.


最后，`ENV`也能用来设置常用的版本号，这样版本的变化变得容易维护了，如下所示：
Lastly, `ENV` can also be used to set commonly used version numbers so that
version bumps are easier to maintain, as seen in the following example:

    ENV PG_MAJOR 9.3
    ENV PG_VERSION 9.3.4
    RUN curl -SL http://example.com/postgres-$PG_VERSION.tar.xz | tar -xJC /usr/src/postgress && …
    ENV PATH /usr/local/postgres-$PG_MAJOR/bin:$PATH

类似于在程序中有常量而不是硬编码的值，这种方法让你能够通过修改一个单一的`ENV`指令来
自动对你容器的中版本进行升级。
Similar to having constant variables in a program (as opposed to hard-coding
values), this approach lets you change a single `ENV` instruction to
auto-magically bump the version of the software in your container.

### ADD or COPY

[Dockerfile手册中的ADD指令](../../reference/builder.md#add)<br/>
[Dockerfile手册中的COPY指令](../../reference/builder.md#copy)
[Dockerfile reference for the ADD instruction](../../reference/builder.md#add)<br/>
[Dockerfile reference for the COPY instruction](../../reference/builder.md#copy)

通常来说，尽管`ADD`和`COPY`指令的功能类似的，但是推荐使用`COPY`指令。因为它与`ADD`指令相比
更加透明。`COPY`仅支持将本地文件拷贝到容器中的基本命令，而`ADD`命令有一些特性（像本地的tar包解压缩
和远程URL的支持）是不明显的。因此，对`ADD`来说最好的使用场景是本地tar包的解压缩到镜像中，例如
`ADD rootfs.tar.xz /`。
Although `ADD` and `COPY` are functionally similar, generally speaking, `COPY`
is preferred. That’s because it’s more transparent than `ADD`. `COPY` only
supports the basic copying of local files into the container, while `ADD` has
some features (like local-only tar extraction and remote URL support) that are
not immediately obvious. Consequently, the best use for `ADD` is local tar file
auto-extraction into the image, as in `ADD rootfs.tar.xz /`.

如果你有多个`Dockerfile`步骤，使用了上下文中不同的文件，对每个文件单独使用`COPY`指令而不是一次性拷贝
所有的文件。这会保证每一步骤的缓存只有在当前步骤的文件发生修改时，才会失效。
If you have multiple `Dockerfile` steps that use different files from your
context, `COPY` them individually, rather than all at once. This will ensure that
each step's build cache is only invalidated (forcing the step to be re-run) if the
specifically required files change.

例如
For example:

    COPY requirements.txt /tmp/
    RUN pip install --requirement /tmp/requirements.txt
    COPY . /tmp/

对`RUN`步骤来说，与将指令`COPY . /tmp/`放在`RUN`指令之前相比，会导致更少的缓存失效
Results in fewer cache invalidations for the `RUN` step, than if you put the
`COPY . /tmp/` before it.

由于镜像大小很重要，不推荐使用`ADD`指令来从远程URL地址获取软件包；你应该使用`curl`或者`wget`。
这样你就可以在解压缩之后，删除你不再需要的文件以及你不需要在你的镜像中增加另外一层。例如你应该避免这样做：
Because image size matters, using `ADD` to fetch packages from remote URLs is
strongly discouraged; you should use `curl` or `wget` instead. That way you can
delete the files you no longer need after they've been extracted and you won't
have to add another layer in your image. For example, you should avoid doing
things like:

    ADD http://example.com/big.tar.xz /usr/src/things/
    RUN tar -xJf /usr/src/things/big.tar.xz -C /usr/src/things
    RUN make -C /usr/src/things all

而是应该这样做：
And instead, do something like:

    RUN mkdir -p /usr/src/things \
        && curl -SL http://example.com/big.tar.xz \
        | tar -xJC /usr/src/things \
        && make -C /usr/src/things all

对于其他项目（文件，目录），如果不需要`ADD`命令的自动解压缩功能，你应该始终使用`COPY`命令。
For other items (files, directories) that do not require `ADD`’s tar
auto-extraction capability, you should always use `COPY`.

### ENTRYPOINT

[Dockerfile手册中的ENTRYPOINT指令](../../reference/builder.md#entrypoint)
[Dockerfile reference for the ENTRYPOINT instruction](../../reference/builder.md#entrypoint)

对于`ENTRYPOINT`来说，最佳的目的是设置镜像的主要执行命令，运行这个镜像看起来就像是这个命令一
样(然后使用命令`CMD`来添加默认的参数)
The best use for `ENTRYPOINT` is to set the image's main command, allowing that
image to be run as though it was that command (and then use `CMD` as the
default flags).

让我们以一个命令行工具`s3cmd`镜像的例子开始：
Let's start with an example of an image for the command line tool `s3cmd`:

    ENTRYPOINT ["s3cmd"]
    CMD ["--help"]

现在这个镜像可以像如下这样执行来显示这个命令的帮助内容：
Now the image can be run like this to show the command's help:

    $ docker run s3cmd

或者使用一个正确是参数来执行一个命令：
Or using the right parameters to execute a command:

    $ docker run s3cmd ls s3://mybucket

这是有用的，由于镜像的名称可以被当作是对可执行文件的引用，例如上面提到的例子。
This is useful because the image name can double as a reference to the binary as
shown in the command above.

`ENTRYPOINT`指令也能和帮助脚本结合进行使用，允许该命令以一种与上述命令相似的方式起作用，即使启动
这个工具需要多个步骤。
The `ENTRYPOINT` instruction can also be used in combination with a helper
script, allowing it to function in a similar way to the command above, even
when starting the tool may require more than one step.

举例而言，[Postgres官方镜像](https://hub.docker.com/_/postgres/)使用下面的脚本来作为其
`ENTRYPOINT`：
For example, the [Postgres Official Image](https://hub.docker.com/_/postgres/)
uses the following script as its `ENTRYPOINT`:

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
> **Note**:
> This script uses [the `exec` Bash command](http://wiki.bash-hackers.org/commands/builtin/exec)
> so that the final running application becomes the container's PID 1. This allows
> the application to receive any Unix signals sent to the container.
> See the [`ENTRYPOINT`](../../reference/builder.md#entrypoint)
> help for more details.


帮助脚本被拷贝进容器中，并通过`ENTRYPOINT`命令在容器启动时执行。
The helper script is copied into the container and run via `ENTRYPOINT` on
container start:

    COPY ./docker-entrypoint.sh /
    ENTRYPOINT ["/docker-entrypoint.sh"]

这个脚本允许用户通过几钟方式跟Postgrs进行交互。
This script allows the user to interact with Postgres in several ways.

它能简单地启动Postgres：
It can simply start Postgres:

    $ docker run postgres

或者，它能被用来执行Postgres并将参数传递给服务器：
Or, it can be used to run Postgres and pass parameters to the server:

    $ docker run postgres postgres --help

最后，它还能被用来执行一个完全不同的工具，例如Bash：
Lastly, it could also be used to start a totally different tool, such as Bash:

    $ docker run --rm -it postgres bash

### VOLUME

[Dockerfile手册中的VOLUME命令](../../reference/builder.md#volume)
[Dockerfile reference for the VOLUME instruction](../../reference/builder.md#volume)

`VOLUME`指令被用来暴露任何数据库存储的区域，配置存储，或者由你的docker容器创建的文件或文件夹。强烈
建议你对镜像中可变的或者对用户提供服务的部分使用`VOLUME`指令。
The `VOLUME` instruction should be used to expose any database storage area,
configuration storage, or files/folders created by your docker container. You
are strongly encouraged to use `VOLUME` for any mutable and/or user-serviceable
parts of your image.

### USER

[Dockerfile手册中的USER指令](../../reference/builder.md#user)
[Dockerfile reference for the USER instruction](../../reference/builder.md#user)

如果一个服务可以在没有特权的情况下运行，使用`USER`指令来修改为一个非root用户。在使用该指令前
通过`RUN groupadd -r postgres && useradd -r -g postgres postgres`来创建用户和组。
If a service can run without privileges, use `USER` to change to a non-root
user. Start by creating the user and group in the `Dockerfile` with something
like `RUN groupadd -r postgres && useradd -r -g postgres postgres`.

> **注意：** 镜像中的用户和组会被分配一个不确定的
> UID或者GID，由于不管镜像是否被重新构建，“下一个”UID或者GID都会被分配。
> 因此，如果UID或者GID是关键点，你应该显式分配它们。
> **Note:** Users and groups in an image get a non-deterministic
> UID/GID in that the “next” UID/GID gets assigned regardless of image
> rebuilds. So, if it’s critical, you should assign an explicit UID/GID.

由于`sudo`有不可预测的TTY或者信号转发行为，这可能会导致更多的问题，因此你应该避免安装或者
使用该命令。如果你必须使用跟`sudo`功能类似的命令(例如，将守护进程初始化为root用户，但是执行时变为非root用户)，
你可以考虑使用["gosu"](https://github.com/tianon/gosu)。
You should avoid installing or using `sudo` since it has unpredictable TTY and
signal-forwarding behavior that can cause more problems than it solves. If
you absolutely need functionality similar to `sudo` (e.g., initializing the
daemon as root but running it as non-root), you may be able to use
[“gosu”](https://github.com/tianon/gosu).

最后，为了避免减少层的次数和复杂度，避免来回切换使用`USER`命令。
Lastly, to reduce layers and complexity, avoid switching `USER` back
and forth frequently.

### WORKDIR

[Dockerfile手册中的WORKDIR指令](../../reference/builder.md#workdir)
[Dockerfile reference for the WORKDIR instruction](../../reference/builder.md#workdir)

为了追求清晰和可读性，你应该总是在`WORKDIR`中使用绝对的路径。同时，你应该使用`WORKDIR`
而不是繁复的类似`RUN cd … && do-something`这样难以阅读，难以错误定位，难以维护的命令。
For clarity and reliability, you should always use absolute paths for your
`WORKDIR`. Also, you should use `WORKDIR` instead of  proliferating
instructions like `RUN cd … && do-something`, which are hard to read,
troubleshoot, and maintain.

### ONBUILD

[Dockerfile手册中的ONBUILD指令](../../reference/builder.md#onbuild)
[Dockerfile reference for the ONBUILD instruction](../../reference/builder.md#onbuild)

`ONBUILD`命令是在当前`Dockerfile`构建执行完毕之后再执行的。
`ONBUILD`是在任何使用`FROM`命令从当前镜像派生的子镜像中执行的。将`ONBUILD`看作是父`Dockerfile`
交给子`Dockerfile`执行的命令。
An `ONBUILD` command executes after the current `Dockerfile` build completes.
`ONBUILD` executes in any child image derived `FROM` the current image.  Think
of the `ONBUILD` command as an instruction the parent `Dockerfile` gives
to the child `Dockerfile`.

一次Docker构建，会在执行任何子`Dockerfile`命令之前执行`ONBUILD`命令。
A Docker build executes `ONBUILD` commands before any command in a child
`Dockerfile`.

`ONBUILD`在`FROM`一个给定的镜像下构建新的镜像是有用的。例如，你会在一个语言栈中使用`ONBUILD`
命令，这个镜像是用来构建任意以该语言编写的软件的`Dockerfile`而准备的，例如，你可以参考
[Ruby的`ONBUILD`变体](https://github.com/docker-library/ruby/blob/master/2.1/onbuild/Dockerfile)
`ONBUILD` is useful for images that are going to be built `FROM` a given
image. For example, you would use `ONBUILD` for a language stack image that
builds arbitrary user software written in that language within the
`Dockerfile`, as you can see in [Ruby’s `ONBUILD` variants](https://github.com/docker-library/ruby/blob/master/2.1/onbuild/Dockerfile).

从`ONBUILD`构建的镜像应该拿到一个有区分的tag，例如`ruby:1.9-onbuild` 或者 `ruby:2.0-onbuild`。

Images built from `ONBUILD` should get a separate tag, for example:
`ruby:1.9-onbuild` or `ruby:2.0-onbuild`.

当你需要在`ONBUILD`中放入`ADD`或者`COPY`指令时，请务必谨慎。由于如果新的构建的上下文没有被添加的
资源，则"onbuild"的镜像会灾难性地失败。像上面推荐的那样，增加一个易于区分的tag，将通过允许`Dockerfile`的作者
作出选择来减少这种情况。
Be careful when putting `ADD` or `COPY` in `ONBUILD`. The “onbuild” image will
fail catastrophically if the new build's context is missing the resource being
added. Adding a separate tag, as recommended above, will help mitigate this by
allowing the `Dockerfile` author to make a choice.

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
