---
description: 学习支持存储驱动的基础概念和技术技术  
keywords: container, storage, driver, AUFS, btfs, devicemapper,zvfs
title: 理解 镜像、容器和存储驱动
redirect_from:
- /en/latest/terms/layer/
---

想要正确和高效的使用和配置存储驱动，首先要理解Docker是如何构建和存储镜像，然后你需要理解容器是融合使用这些镜像的，最后你需要镜像和容器操作的的技术介绍。

## 镜像和镜像的层

每个Docker镜像一组只读的镜像的层组成的，每一个镜像的层描述了对文件系统做了哪些变化。镜像互相堆叠到别的镜像的上面，最终形成容器的根文件系统，下面这张图展示了Ubuntu 15.04的镜像的4层的镜像层的栈。

![](images/image-layers.jpg)

Docker的存储驱动负责管理层的堆叠关系，并最终将镜像的层组成一个一致的文件系统。

当你创建一个新的容器时，你将在镜像的层的这个栈上面添加一个新的、可写的层，我们通常管这一层叫做"容器层"，所有的对这个运行中容器的修改，比如说添加新的文件、修改已经存在的文件或者删除文件，都会写如到这个可写的"容器层"，下面这个图展示了基于Ubuntu 15.04的容器的层信息。

![](images/container-layers.jpg)

### 内容可寻址的存储

Docker 1.10 引入了新的内容可寻址的存储模型，它是一种全新的在磁盘上寻址镜像及镜像层的方式。在这个范式值钱啊，镜像和层是通过随机的一个UUID去存储和引用的，在新的模型中这个被替换成安全的*内容哈希*。

新的模型提升了安全性，避免了ID冲突，并保证了在pull、push、load和save操作时的数据的完整性。并且通过内容哈希寻址，提供了更好的镜像层的共享，就算两个镜像并不是同一个构建产生的，只要内容哈希相同，同样会共享层的存储。

下面这张图对比之前的图，箭头指出的变化就是Docker 1.10做了的增强。

![](images/container-layers-cas.jpg)

如你所见，所有的镜像层的ID是通过密码学哈希计算出来的，不过容器的ID还是随机计算出的UUID。

一下是关于新的模型需要注意的一些内容，包括：

1. 已有的镜像迁移
2. 镜像和层的文件系统结构

已有的镜像，他们是通过更早之前版本的Docker创建或者pull下来的，在使用新的模型的Docker中时需要对原有的镜像模型进行迁移。这个迁移会在你升级到新的Docker daemon启动时进行，将对原有的镜像的层进行重新的计算哈希。当迁移完成后，所有的镜像就通过新的存储结构来存储，都会被记录上安全的ID了。

虽然这个迁移的过程是自动并透明的，不过需要大量集中的计算来完成。所以这个过程会消耗大量的时间，计算过程中Docker daemon不会响应请求.

有一个迁移的工具帮助你在升级Docker之前就迁移已有镜像到新的存储格式，使用这个工具意味着在迁移数据不是非要升级Docker后Docker daemon启动时才进行，这样就可以避免升级的时候因迁移而长时间的停止服务。通过它你可以将已有的镜像事先迁移到新的存储模型上，然后把将迁移的结果放到你的环境中的其他已经升级到新版本的Docker Daemon上。

Docker, Inc.提供了这个迁移工具，可以通过一个容器直接run起来去执行迁移，你可以通过[https://github.com/docker/v1.10-migrator/releases](https://github.com/docker/v1.10-migrator/releases)去下载这个工具。

你需要将Docker在主机上的数据目录映射到这个"migrator"容器中，如果你正在用默认的Docker的数据目录，那么run这个迁移容器的命令就是像下面这样：

    $ sudo docker run --rm -v /var/lib/docker:/var/lib/docker docker/v1.10-migrator

如果你正在使用`devicemapper`的存储驱动，你还需要在run迁移容器的时候增加`--privileged`参数，以便这个容器能够访问到你的存储的设备。

#### 迁移示例

下面的示例将演示如何在Docker daemon版本是1.9.1并使用AUFS作为存储驱动的宿主机上使用迁移工具，这个宿主机是一个**t2.micro** 规格的AWS EC2的的实例，他的配置是1个vCPU，1GB的内存并挂载了一个8GB的SSD EBS数据盘。Docker的数据目录(`/var/lib/docker`)占用了2GB的磁盘空间。

    $ docker images

    REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
    jenkins             latest              285c9f0f9d3d        17 hours ago        708.5 MB
    mysql               latest              d39c3fa09ced        8 days ago          360.3 MB
    mongo               latest              a74137af4532        13 days ago         317.4 MB
    postgres            latest              9aae83d4127f        13 days ago         270.7 MB
    redis               latest              8bccd73928d9        2 weeks ago         151.3 MB
    centos              latest              c8a648134623        4 weeks ago         196.6 MB
    ubuntu              15.04               c8be1ac8145a        7 weeks ago         131.3 MB

    $ sudo du -hs /var/lib/docker

    2.0G    /var/lib/docker

    $ time docker run --rm -v /var/lib/docker:/var/lib/docker docker/v1.10-migrator

    Unable to find image 'docker/v1.10-migrator:latest' locally
    latest: Pulling from docker/v1.10-migrator
    ed1f33c5883d: Pull complete
    b3ca410aa2c1: Pull complete
    2b9c6ed9099e: Pull complete
    dce7e318b173: Pull complete
    Digest: sha256:bd2b245d5d22dd94ec4a8417a9b81bb5e90b171031c6e216484db3fe300c2097
    Status: Downloaded newer image for docker/v1.10-migrator:latest
    time="2016-01-27T12:31:06Z" level=debug msg="Assembling tar data for 01e70da302a553ba13485ad020a0d77dbb47575a31c4f48221137bb08f45878d from /var/lib/docker/aufs/diff/01e70da302a553ba13485ad020a0d77dbb47575a31c4f48221137bb08f45878d"
    time="2016-01-27T12:31:06Z" level=debug msg="Assembling tar data for 07ac220aeeef9febf1ac16a9d1a4eff7ef3c8cbf5ed0be6b6f4c35952ed7920d from /var/lib/docker/aufs/diff/07ac220aeeef9febf1ac16a9d1a4eff7ef3c8cbf5ed0be6b6f4c35952ed7920d"
    <snip>
    time="2016-01-27T12:32:00Z" level=debug msg="layer dbacfa057b30b1feaf15937c28bd8ca0d6c634fc311ccc35bd8d56d017595d5b took 10.80 seconds"

    real    0m59.583s
    user    0m0.046s
    sys     0m0.008s
在`docker run`之前预先使用Unix的`time`命令记录整个操作的时间，如你所见，整个过程消耗了大约2分钟的时间迁移了占用磁盘2GB的7个Docker镜像。其实这个过程中还包含了pull `docker/v1.10-migrator`镜像的时间(大约是3.5秒)，而通过40个vCPU、160GB内存和8GB的增强IO型的EBS数据盘的m4.10xlarge的规格的EC2实例测试的话，时间效率提升如下：


    real    0m9.871s
    user    0m0.094s
    sys     0m0.021s

这表明了，执行迁移操作的宿主机的硬件的配置会直接影响到迁移操作的效率。

## 容器和层

容器和镜像的主要的区别在于最上层是否是可写的。所有对容器的修改都会在容器的可写层中增加或修改。当一个容器被删除时，这层可写层也同时被删除，但这层可写层下面的镜像的层会保持不变。

因为每个容器都有自己的可写的容器层，容器中所有的修改被保存在这层可写层中，这样就保证了多个容器可以共享同样的下层的镜像，并在容器中又可以有自己的数据状态。下面这张图展示了多个容器共享Ubuntu 15.04的镜像。

![](images/sharing-layers.jpg)

Docker存储驱动的职责是同时管理Docker的镜像的层和容器的可写层。不同的Docker存储驱动对这个职责有着不同的实现方式，实现这个职责的关键技术对镜像分层的支持技术，以及copy-on-write(CoW)。

## Copy-on-write策略

共享是优化资源占用的一个好方法，人们在日常生活中会本能的共享资源。例如，双胞胎Jane和Joseph在不同的时间通过不同的老师学习线性代数的课程，不过他们使用同一个的练习册。现在Jane需要完成练习册上第11页的作业，Jane就可以从这个共同的练习册复制出第11页的内容，完成这个家庭作业，原本的联系册并没有变化，只是Jane有修改了11页的原本练习册的一个副本而已。

Copy-on-write 是一个关于共享和复制的类似的策略，在这种策略中，系统的不同进程在使用同样数据时会共享同一个数据的实例，而不是每个进程有自己的一个数据的副本。这个时候如果某个进程需要修改或者写入到那个数据中，操作系统才会将这份数据拷贝一份副本供这个进程使用，并且这个副本只允许那个需要修改或写入的进程访问，其他的进程则继续使用原始的那份数据。

Docker在镜像和容器的实现中都使用到了这个copy-on-write的技术，Docker通过这个Copy-on-write的策略优化了镜像和容器的磁盘占用，以及Docker容器启动的时间。下一章节我们将介绍Docker是如何在容器和镜像的使用copy-on-write来控制资源共享和拷贝。

### 通过共享优化镜像尺寸

下面的章节讨论镜像层和copy-on-write技术。 所有镜像和容器层都存在于Docker主机的 *本地存储* 中，并由存储驱动管理。 在基于Linux的Docker主机上，这通常位于`/var/lib/docker/`目录下。

Docker 客户端在pull和push镜像的时候会展示出镜像的层，例如当我们从Docker Hub pull `ubuntu:15.04`的Docker 镜像：

    $ docker pull ubuntu:15.04

    15.04: Pulling from library/ubuntu
    1ba8ac955b97: Pull complete
    f157c4e5ede7: Pull complete
    0b7e98f84c4c: Pull complete
    a3ed95caeb02: Pull complete
    Digest: sha256:5e279a9df07990286cce22e1b0f5b0490629ca6d187698746ae5e28e604a640e
    Status: Downloaded newer image for ubuntu:15.04

通过输出，你可以看到这个命令实际上pull了4层的镜像层，上面的每一行罗列了镜像的层和层的UUID值或者hash值，镜像的这四层最终组成`ubuntu:15.04`的镜像。

这些层保存在Docker宿主机的本地存储中，每一层在存储中有自己单独的目录。

Docker 1.10的版本之前的Docker会保存镜像的层到镜像层ID同名的目录下面，然而，这个规则并不适用于Docker 1.10及以后的版本。比如，下面的命令是从Docker Hub pull一个镜像，然后在1.9.1的版本的Docker Engine，可以看到下面的目录结构。

    $  docker pull ubuntu:15.04

    15.04: Pulling from library/ubuntu
    47984b517ca9: Pull complete
    df6e891a3ea9: Pull complete
    e65155041eed: Pull complete
    c8be1ac8145a: Pull complete
    Digest: sha256:5e279a9df07990286cce22e1b0f5b0490629ca6d187698746ae5e28e604a640e
    Status: Downloaded newer image for ubuntu:15.04

    $ ls /var/lib/docker/aufs/layers

    47984b517ca9ca0312aced5c9698753ffa964c2015f2a5f18e5efa9848cf30e2
    c8be1ac8145a6e59a55667f573883749ad66eaeef92b4df17e5ea1260e2d7356
    df6e891a3ea9cdce2a388a2cf1b1711629557454fd120abd5be6d32329a0e0ac
    e65155041eed7ec58dea78d90286048055ca75d41ea893c7246e794389ecf203

注意到，这四个文件夹的名字就是下载镜像时显示的层ID，然后我们在运行1.10的Docker Engine版本的机器上测试同样的操作。

    $ docker pull ubuntu:15.04
    15.04: Pulling from library/ubuntu
    1ba8ac955b97: Pull complete
    f157c4e5ede7: Pull complete
    0b7e98f84c4c: Pull complete
    a3ed95caeb02: Pull complete
    Digest: sha256:5e279a9df07990286cce22e1b0f5b0490629ca6d187698746ae5e28e604a640e
    Status: Downloaded newer image for ubuntu:15.04

    $ ls /var/lib/docker/aufs/layers/
    1d6674ff835b10f76e354806e16b950f91a191d3b471236609ab13a930275e24
    5dbb0cbe0148cf447b9464a358c1587be586058d9a4c9ce079320265e2bb94e7
    bef7199f2ed8e86fa4ada1309cfad3089e0542fec8894690529e4c04a7ca2d73
    ebf814eccfe98f2704660ca1d844e4348db3b5ccc637eb905d4818fbfb00a06a

看到文件夹并不匹配于上一步的pull的时候显示的层ID。

不管在1.10的Docker版本之前，还是之后对Docker的版本的变化，Docker都是会在镜像间共享层的。例如，比如你`pull`了一个和之前pull过的镜像共享镜像层的镜像，Docker daemon就会识别到这个共享，并且只pull之前没有pull过的镜像的层。在第二次pull的时候，这第二个镜像会共享任何相同的镜像层。

你现在可以自己实验下，通过刚才pull的`ubuntu:15.04`的镜像开始，在上面做一些修改，然后构建一个新的镜像。可以通过使用`Dockerfile`文件以及`docker build`命令做修改和构建。

1. 在一个空的目录，创建Dockerfile文件，并且From ubuntu:15.04的镜像

        FROM ubuntu:15.04

2. 在镜像的`/tmp`目录创建文件并写入"Hello world"。

    当你完成后，Dockerfile应该是如下两行：

        FROM ubuntu:15.04

        RUN echo "Hello world" > /tmp/newfile

3. 保存和关闭文件

4. 在和你的`Dockerfile`文件夹同样目录的终端中运行如下命令：


        $ docker build -t changed-ubuntu .

        Sending build context to Docker daemon 2.048 kB
        Step 1 : FROM ubuntu:15.04
         ---> 3f7bcee56709
        Step 2 : RUN echo "Hello world" > /tmp/newfile
         ---> Running in d14acd6fad4e
         ---> 94e6b7d2c720
        Removing intermediate container d14acd6fad4e
        Successfully built 94e6b7d2c720

    > **Note:** 这个命令中的执行目录的参数"."很重要. 它表示`docker build`命令使用
    >  当前的工作目录作为它的构建的上下文。

    下面的输出展示了新的镜像以及它的ID `94e6b7d2c720`.

5. 运行`docker images`命令验证新的`changed-ubuntu`镜像是存在在Docker宿主机的本地存储的。
   

        REPOSITORY       TAG      IMAGE ID       CREATED           SIZE
        changed-ubuntu   latest   03b964f68d06   33 seconds ago    131.4 MB
        ubuntu           15.04    013f3d01d247   6 weeks ago       131.3 MB

6. 运行`docker history`命令看到在创建新的`changed-ubuntu`用到了哪些镜像的层。
   
        $ docker history changed-ubuntu
        IMAGE               CREATED              CREATED BY                                      SIZE        COMMENT
        94e6b7d2c720        2 minutes ago       /bin/sh -c echo "Hello world" > /tmp/newfile    12 B
        3f7bcee56709        6 weeks ago         /bin/sh -c #(nop) CMD ["/bin/bash"]             0 B
        <missing>           6 weeks ago         /bin/sh -c sed -i 's/^#\s*\(deb.*universe\)$/   1.879 kB
        <missing>           6 weeks ago         /bin/sh -c echo '#!/bin/sh' > /usr/sbin/polic   701 B
        <missing>           6 weeks ago         /bin/sh -c #(nop) ADD file:8e4943cd86e9b2ca13   131.3 MB

	`docker history`输出中可以看到最上层有一个新的`94e6b7d2c720`镜像层。这个是由Dockerfile中的 `echo "Hello world" > /tmp/newfile`命令创建的。而下面的4层是和组成`ubuntu:15.04`的镜像4层是相同的。

> **Note:** 在Docker 1.10的内容可寻址的模型下，镜像的历史数据不再保存在每个层的配置文件中。
> 它现在存储在与整个镜像关联的一个配置文件中。这导致了有一些镜像在执行`docker history`命令时显示出"missing"，这个是正常的可以忽略的行为。
>
> 你可能听说过类似的镜像，称为 *flat images*.

可以注意到，新的镜像`changed-ubuntu`并没有对镜像层都做拷贝。可以看下面这张图，新的镜像与`ubuntu:15.04`镜像共享了底下的几层。

![](images/saving-space.jpg)

`docker history`同样展示了每一层镜像层的大小信息。你可以看到，`94e6b7d2c720 `层只占用了12个字节的磁盘空间。这意味着我们创建的`changed-ubuntu`镜像只消耗二外的12字节的机器磁盘空间，所有的`94e6b7d2c720`下面的层已经在宿主机上存在，并和别的镜像共享。

这里的镜像层的共享使得Docker的镜像和容器有很好的空间效率。

### Copy提升容器的效率

你之前了解到容器即使镜像加上一个读写层。下面的图展示了基于`ubuntu:15.04`创建的容器的层：

![](images/container-layers-cas.jpg)

所有的对容器的修改都会保存在容器的可读写层中。其他的镜像的层是只读的层，不可以被修改。这样就意味着多个容器可以安全的共享一个底层的镜像。下面这个图展示了多个容器共享一份`ubuntu:15.04`的镜像实例。每个容器拥有自己的读写层，而他们都共享同一份ubuntu:15.04的镜像实例。

![](images/sharing-layers.jpg)

当修改一个镜像中已经存在的文件时，Docker使用存储驱动执行一个copy-on-write操作。这个操作依赖于存储驱动是如何实现的。例如对于AUFS和OverlayFS存储驱动来说，这个copy-on-write操作的流程如下：

* 搜索需要修改的文件所在的镜像的层。这个过程依次从最新的最上层的层到最下面的镜像层。
* 当找到这个文件时，执行一个"copy-up"的操作。一个"copy up"拷贝文件到容器自己的读写层中。
* 修改容器的读写层中这个文件的拷贝。

Brtfs，ZFS以及其他的存储驱动处理这个copy-on-write要复杂一些，你可以后面在他们的更细节的介绍中了解到他们的copy-on-write方法。

容器拷贝写入大量数据，会消耗比容器写入数据更多的磁盘空间。这是因为大部分写入的操作会导致在容器读写层产生一块新的空间。如果你的容器需要写入大量的数据，你应该考虑使用数据卷。

一个copy-up操作会显著的增加性能的消耗。这个消耗不同的存储驱动消耗也不尽相同。不过，大的文件，多的文件，以及更深的文件夹会让这个消耗更显著。幸运的是这个操作只会在第一次修改文件时进行。后续的对用一个文件的修改就不会触发copy-up操作，而会直接修改容器在容器读写层中已存在的实例。

让我们看下我们如果通过我们之前build的`changed-ubuntu`启动5个容器会发生什么：

1. 在你的Docker宿主机的终端上执行5次的docker run启动容器

        $ docker run -dit changed-ubuntu bash

        75bab0d54f3cf193cfdc3a86483466363f442fba30859f7dcd1b816b6ede82d4

        $ docker run -dit changed-ubuntu bash

        9280e777d109e2eb4b13ab211553516124a3d4d4280a0edfc7abf75c59024d47

        $ docker run -dit changed-ubuntu bash

        a651680bd6c2ef64902e154eeb8a064b85c9abf08ac46f922ad8dfc11bb5cd8a

        $ docker run -dit changed-ubuntu bash

        8eb24b3b2d246f225b24f2fca39625aaad71689c392a7b552b78baf264647373

        $ docker run -dit changed-ubuntu bash

        0ad25d06bdf6fca0dedc38301b2aff7478b3e1ce3d1acd676573bba57cb1cfef

	这样启动了5个基于`changed-ubuntu`镜像的容器。当每个容器被创建时，Docker增加一个读写层并分配一个随机的UUID。即`docker run`返回的数值。

2. 使用`docker ps`命令验证5个容器都在运行中

        $ docker ps
        CONTAINER ID    IMAGE             COMMAND    CREATED              STATUS              PORTS    NAMES
        0ad25d06bdf6    changed-ubuntu    "bash"     About a minute ago   Up About a minute            stoic_ptolemy
        8eb24b3b2d24    changed-ubuntu    "bash"     About a minute ago   Up About a minute            pensive_bartik
        a651680bd6c2    changed-ubuntu    "bash"     2 minutes ago        Up 2 minutes                 hopeful_turing
        9280e777d109    changed-ubuntu    "bash"     2 minutes ago        Up 2 minutes                 backstabbing_mahavira
        75bab0d54f3c    changed-ubuntu    "bash"     2 minutes ago        Up 2 minutes                 boring_pasteur

	上面的输出表示有5个运行中的容器，都共享了`changed-ubuntu`的镜像。每个`CONTAINER ID`是在创建容器的时候产生的UUID。

3. 列出本地磁盘上的容器目录

        $ sudo ls /var/lib/docker/containers

        0ad25d06bdf6fca0dedc38301b2aff7478b3e1ce3d1acd676573bba57cb1cfef
        9280e777d109e2eb4b13ab211553516124a3d4d4280a0edfc7abf75c59024d47
        75bab0d54f3cf193cfdc3a86483466363f442fba30859f7dcd1b816b6ede82d4
        a651680bd6c2ef64902e154eeb8a064b85c9abf08ac46f922ad8dfc11bb5cd8a
        8eb24b3b2d246f225b24f2fca39625aaad71689c392a7b552b78baf264647373

Docker的copy-on-write策略并不仅仅较少了容器磁盘空间的开销，它还在启动容器的时候降低了时间的开销。下面这个图展示了5个容器共享了同一个只读的`changed-ubuntu`镜像的实例。

![](images/shared-uuid.jpg)

如果Docker如果在启动新容器的时候坐所有的底层镜像的拷贝，那么容器启动会非常漫长，并且磁盘使用也会暴涨了。

## 数据卷和存储驱动

当一个容器删除后，除了存储到*数据卷*中的数据，其他所有在容器中的数据将被删除。

一个数据卷在Docker宿主机的文件系统上是一个目录或者文件，通过挂载到容器内部的文件系统。数据卷并不归存储驱动管理。读取和写入到数据卷会拥有和宿主机上同样的效率。你可以挂载任意数量的数据卷到容器中，多个容器也可以共享一个或者多个数据卷。

下面的图展示了一个运行着两个容器的Docker宿主机。每个容器都在宿主机的本地存储(`/var/lib/docker/...`)下有自己的一块地址空间。而一个共享的数据卷位于宿主机的`/data`目录，被直接挂载到这些容器中。

![](images/shared-volume.jpg)

数据卷属于在Docker宿主机上的外部存储，对数据卷中数据的修改独立于存储驱动的控制。当一个容器删除的时候，在数据卷中保存的数据不会被删除。

更多关于数据卷的信息可以参考
[管理容器中的数据](/engine/tutorials/dockervolumes/).

## 相关信息

* [选择存储驱动](selectadriver.md)
* [AUFS存储驱动实践](aufs-driver.md)
* [Btrfs 存储驱动实践](btrfs-driver.md)
* [Device Mapper 存储驱动实践](device-mapper-driver.md)
