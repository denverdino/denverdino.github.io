---
description: Learn how to optimize your use of ZFS driver.
keywords: 'container, storage, driver, ZFS '
title: Docker and ZFS in practice
---

ZFS是新一代文件系统。支持许多高级存储技术（如卷管理，快照，校验，压缩和重复数据删除，复制等）。

它由Sun Microsystems（现为Oracle Corporation）创建，并且是在CDDL开源许可下的。
由于CDDL和GPL之间的许可不兼容性，ZFS不能作为主线Linux内核的一部分提供。
但是，ZFS在Linux（ZoL）项目提供了一个版本树之外的内核模块和用户空间工具。用户可以单独安装。

Linux上的ZFS（ZoL）接口是健康和成熟的。
但是，在这个时间点，不建议使用`zfs`Docker存储驱动程序在生产环境中使用，除非您在Linux上拥有丰富的ZFS使用经验。

> **Note:** 在Linux平台上还有一个ZFS的FUSE实现。它应该能和Docker一起工作，但不推荐。
> 原生ZFS驱动程序（ZoL）经过了更多的测试，性能更高，使用更广泛。本文档的其余部分将涉及ZoL原生接口。


## 镜像分层和ZFS共享

Docker`zfs`存储驱动程序大量使用三个ZFS数据集：

- filesystems
- snapshots
- clones

ZFS文件系统进行精简配置，并通过按需分配的方式从ZFS池（zpool）为其分配空间。
快照和克隆是为了ZFS文件系统的节省空间的时间点而创建的副本。
其中快照是只读的，克隆是可读写的。只能从快照创建克隆。这个简单的关系如下图所示。

![](images/zfs_clones.jpg)

图中的实线显示了创建一个克隆的流程。步骤1创建文件系统的快照，步骤2从快照创建克隆。
虚线通过快照显示克隆和文件系统之间的关系。所有三个ZFS数据集从相同的底层zpool中抽取空间。

在使用了`zfs`存储驱动程序的Docker宿主机上，镜像的基本层是ZFS文件系统。
每个子层是基于其下层的ZFS快照的克隆。容器是基于从其创建的镜像的顶层的ZFS快照的克隆。
所有ZFS数据集都从公共zpool中抽取其空间。下图显示了如何将它与基于两层镜像的正在运行的容器组合在一起。

![](images/zfs_zpool.jpg)

以下过程说明如何分层镜像和创建容器。该过程基于上图。

1. 镜像的基本层作为ZFS文件系统存在于Docker宿主机上。

    这个文件系统消耗来自zpool的空间，用于在`/var/lib/docker`创建Docker宿主机的本地存储区域。

2. 附加的镜像层是托管在其正下方的镜像层数据集的克隆。

    在图中，通过创建基本层的ZFS快照，然后从该快照创建克隆，添加“第1层”。
    克隆是可写的，并且从zpool按需消耗空间。快照是只读的，将基础层维护为不可变对象。

3. 当容器启动时，在镜像上方添加一个读写层。

    在上图中，容器的读写层是通过创建镜像顶层的快照（第1层）并从该快照创建克隆的。

    当对容器进行更改时，将通过按需分配操作从zpool为其分配空间。默认情况下，ZFS将以128K的块为单位来按需分配空间。

这个从 *只读* 快照创建子镜像层和容器的过程允许镜像被保持为不可变的对象。

## 使用ZFS对容器读取和写入

使用`zfs`存储驱动程序的容器读取起来非常简单。
新创建的容器是基于ZFS的一个克隆。这个克隆最初和其复制源的数据集共享所有数据。
这意味着使用`zfs`存储驱动程序的读操作很快，即使正在读取的数据尚未复制到容器中。数据块的共享如下图所示。

![](images/zpool_blocks.jpg)

向容器写入新数据是按需分配的。每次容器需要新的区域进行写入时，都会从zpool分配新的块。
这意味着容器写入新数据时需要消耗额外的空间。新空间从底层zpool分配给容器（ZFS Clone）。

在容器中更新 *现有数据* 的实现方式是，通过将新的块分配给容器克隆，并更改数据存储中有变更的部分完成的。
原始块不变，允许底层镜像数据集保持不变。这与写入正常的ZFS文件系统是相同的，并且是写时复制的一种实现。

## 配置Docker使用ZFS存储驱动程序

使用`zfs`存储驱动程序的宿主机的前置条件，`/var/lib/docker`作为ZFS文件系统被挂载。
本节介绍如何在Ubuntu 14.04系统上在Linux（ZoL）上安装和配置原生ZFS。

### 前置依赖

如果您已经在Docker宿主机上使用了Docker Daemon，并且想要保留已有的镜像。
那么在尝试此过程之前，请将已有推送Docker Hub或您的私有Docker Registry中。

停止你的Docker Daemon。然后，确保在`/dev/xvdb`上有一个空闲的块设备。
设备标识符在您的环境中可能有所不同，您应该在安装配置的整个过程中将设备标识符替换您自己的值。

### 在 Ubuntu 16.04 LTS 上安装ZFS

1. 如果Docker Daemon已经在运行了，关闭它。

2. 安装`zfs`的安装包

        $ sudo apt-get install -y zfs

        Reading package lists... Done
        Building dependency tree
        <output truncated>

3. 确认`zfs`模块已经正确装载

        $ lsmod | grep zfs

        zfs                  2813952  3
        zunicode              331776  1 zfs
        zcommon                57344  1 zfs
        znvpair                90112  2 zfs,zcommon
        spl                   102400  3 zfs,zcommon,znvpair
        zavl                   16384  1 zfs

### 在 Ubuntu 14.04 LTS 上安装ZFS

1. 如果Docker Daemon已经在运行了，关闭它。

2. 安装`software-properties-common`安装包

    This is required for the `add-apt-repository` command.

        $ sudo apt-get install -y software-properties-common

        Reading package lists... Done
        Building dependency tree
        <output truncated>

3. 添加`zfs-native`软件包信息。

        $ sudo add-apt-repository ppa:zfs-native/stable

         The native ZFS filesystem for Linux. Install the ubuntu-zfs package.
        <output truncated>
        gpg: key F6B0FC61: public key "Launchpad PPA for Native ZFS for Linux" imported
        gpg: Total number processed: 1
        gpg:               imported: 1  (RSA: 1)
        OK

4. 对所有的注册仓库，更新软件包列表

        $ sudo apt-get update

        Ign http://us-west-2.ec2.archive.ubuntu.com trusty InRelease
        Get:1 http://us-west-2.ec2.archive.ubuntu.com trusty-updates InRelease [64.4 kB]
        <output truncated>
        Fetched 10.3 MB in 4s (2,370 kB/s)
        Reading package lists... Done

5. 安装`ubuntu-zfs`软件包

        $ sudo apt-get install -y ubuntu-zfs

        Reading package lists... Done
        Building dependency tree
        <output truncated>

6. 装载`zfs`模块

        $ sudo modprobe zfs

7. 确认`zfs`模块已经正确装载

        $ lsmod | grep zfs

        zfs                  2768247  0
        zunicode              331170  1 zfs
        zcommon                55411  1 zfs
        znvpair                89086  2 zfs,zcommon
        spl                    96378  3 zfs,zcommon,znvpair
        zavl                   15236  1 zfs

## Docker配置ZFS

一旦ZFS安装装载完成，你就做好了Docker配置ZFS的基本工作


1. 创建一个新的`zpool`

        $ sudo zpool create -f zpool-docker /dev/xvdb

    该命令创建`zpool`，并将其命名为"zpool-docker"。名称是随意的。

2. 确认`zpool`已经存在

        $ sudo zfs list

        NAME            USED  AVAIL    REFER  MOUNTPOINT
        zpool-docker    55K   3.84G    19K    /zpool-docker

3. 创建一个新的挂载点到`/var/lib/docker`

        $ sudo zfs create -o mountpoint=/var/lib/docker zpool-docker/docker

4. 检查前面几步是否正常

        $ sudo zfs list -t all

        NAME                 USED  AVAIL  REFER  MOUNTPOINT
        zpool-docker         93.5K  3.84G    19K  /zpool-docker
        zpool-docker/docker  19K    3.84G    19K  /var/lib/docker

    现在你有一个ZFS文件系统挂载到`/var/lib/docker`，Docker Daemon应该自动加载`zfs`存储驱动程序。

5. 启动Docker Daemon

        $ sudo service docker start

        docker start/running, process 2315

    启动Docker Daemon的过程可能会有所不同，具体取决于您使用的Linux发行版。
    通过将`--storage-driver=zfs`参数传递给`dockerd`命令，或者添加到Docker配置文件中的`DOCKER_OPTS`行，
    都可以强制Docker守护进程以`zfs`存储驱动程序启动。

6. 验证Docker Daemon已经使用了`zfs`存储驱动。

        $ sudo docker info

        Containers: 0
        Images: 0
        Storage Driver: zfs
         Zpool: zpool-docker
         Zpool Health: ONLINE
         Parent Dataset: zpool-docker/docker
         Space Used By Parent: 27648
         Space Available: 4128139776
         Parent Quota: no
         Compression: off
        Execution Driver: native-0.2
        [...]

    上面的命令的输出显示Docker Daemon使用`zfs`存储驱动程序，
    父数据集是之前创建的`zpool-docker/docker`文件系统。

您的Docker宿主机现在正在使用ZFS存储用于管理其镜像和容器。

## 使用ZFS驱动时的Docker性能表现

使用`zfs`存储驱动程序时，有几个因素将影响到Docker的性能。

- **内存**。内存对ZFS性能有重大影响。
回想一个事实，ZFS最初设计用于大型Sun Solaris服务器，其拥有大量的内存。在调整Docker宿主机大小时，请记住这一点。

- **ZFS特性**。使用ZFS功能（如重复数据删除）会显着增加ZFS的内存使用。
出于内存消耗和性能方面的原因，建议您关闭ZFS重复数据删除的功能。
但是，仍然可以在堆栈中其他层（如SAN或NAS阵列）使用重复数据删除，因为这些操作不会影响ZFS内存使用和性能。
如果使用SAN，NAS或其他硬件RAID技术，则应继续遵循在ZFS中使用它们现有的最佳实践。

- **ZFS缓存**。ZFS在内存中高速缓存磁盘块的功能被称为adaptive replacement cache (ARC)。
ZFS *Single Copy ARC* 的特性允许块的单个缓存副本由文件系统中的多个克隆共享。
这意味着多个正在运行的容器可以共享缓存块的单个副本。这意味着ZFS是PaaS和其他高密度使用场景的一个很好选择。

- **碎片**。碎片是像ZFS这样的写时复制文件系统的自然产物。
ZFS使用128K的块用于写入，也会为Cob操作分配 *slabs*（多个128K块），以减少碎片。
ZFS intent log（ZIL）和写入合并（延迟写入）也有助于减少碎片。

- **Linux的原生ZFS驱动程序**。尽管Docker`zfs`存储驱动程序支持ZFS FUSE实现，但对高性能有要求时不推荐使用。
Linux原生ZFS驱动的性能优于FUSE实现。

以下通用的性能最佳实践也适用于ZFS。

- **使用SSD**。为了获得最佳性能，使用快速存储介质（例如固态设备（SSD））总是管用的。
但是，在您的SSD存储有限时，建议优先将ZIL放在SSD上。

- **使用数据卷**。数据量提供了最佳和最可预期的性能。
因为它们绕过了存储驱动程序，并且不会因为精简配置和写时复制而引入的任何潜在的开销。
因为这些原因，您可能会希望让数据卷承担比较大的负载。
