---
description: 学习如何为你的容器选择合适的存储驱动  
keywords: container, storage, driver, AUFS, btfs, devicemapper,zvfs
title: 选择存储驱动  
---

这一篇文章中描述了Docker的存储驱动的特性。它列举了Docker支持的存储驱动以及管理他们的基础的命令。最后这篇文章将提供选择存储驱动的指导。

这篇文章上的内容需要读者事先对[docker存储驱动技术](imagesandcontainers.md)有了解。

## 可插拔的存储驱动架构

Docker拥有一个可插拔的存储驱动架构。这个架构使得你可以灵活地“插入”一个最适配于你的系统的存储驱动。每一个Docker的存储驱动都是基于Linux的文件系统或者数据卷管理。而且每个存储驱动都有自己基于Linux filesystem管理镜像和容器的层的不同实现。这意味着不同的存储驱动将适配不同的场景。

在你决定使用哪种存储驱动时，你只需要在Docker daemon启动的时候设置，之后Docker daemon就只会使用这一个存储驱动，所有的这个docker daemon创建的容器将使用这个存储驱动。下面这张表展示了目前支持的存储驱动依赖的技术和他们的驱动名字：

|技术    |存储驱动名    |
|--------------|-----------------------|
|OverlayFS     |`overlay` or `overlay2`|
|AUFS          |`aufs`                 |
|Btrfs         |`btrfs`                |
|Device Mapper |`devicemapper`         |
|VFS           |`vfs`                  |
|ZFS           |`zfs`                  |

如果想找到目前daemon所设置的存储驱动是哪个，可以通过`docker info`命令：

    $ docker info

    Containers: 0
    Images: 0
    Storage Driver: overlay
     Backing Filesystem: extfs
    Execution Driver: native-0.2
    Logging Driver: json-file
    Kernel Version: 3.19.0-15-generic
    Operating System: Ubuntu 15.04
    ... output truncated ...

上面的`info`子命令展示出当前Docker daemon是使用基于`extfs`后端文件系统的`overlay`存储驱动。`extfs`的值的意思是`overlay`存储驱动是在已有的(ext)文件系统上做镜像和容器的层的操作的。后端文件系统即Docker的在宿主机上创建的存储目录`/var/lib/docker`下的存储区域。

要使用什么样的存储驱动，一定程度上依赖于你计划为Docker的本地存储区域的后端文件系统。一些存储驱动可以在不同的后端文件系统上使用，而有些存储驱动就要求后端存储系统需要是和存储驱动的技术是一样的。比如，`btrfs`存储驱动就需要在Btrfs的后端存储系统。下面这张列表列举了存储驱动和他们对宿主机文件系统的匹配的要求。

|存储驱动 |一般使用的后端存储 |不能使用的后端存储                                         |
|---------------|-----------------|----------------------------------------------------|
|`overlay`      |`ext4` `xfs`     |`btrfs` `aufs` `overlay` `overlay2` `zfs` `eCryptfs`|
|`overlay2`     |`ext4` `xfs`     |`btrfs` `aufs` `overlay` `overlay2` `zfs` `eCryptfs`|
|`aufs`         |`ext4` `xfs`     |`btrfs` `aufs` `eCryptfs`                           |
|`btrfs`        |`btrfs` _only_   |   N/A                                              |
|`devicemapper` |`direct-lvm`     |   N/A                                              |
|`vfs`          |debugging only   |   N/A                                              |
|`zfs`          |`zfs` _only_     |   N/A                                              |


> **Note**
> "不能使用" 意味着一些存储驱动不能在这种后端的格式的文件系统上使用。

你可以通过`dockerd`的`--storage-driver=<name>`命令行参数中设置存储驱动，也可以通过设置`/etc/default/docker`配置文件中的`DOCKER_OPTS`行设置存储驱动。

下面的展示了通过`dockerd`命令和`devicemapper`存储驱动启动Docker daemon：

    $ dockerd --storage-driver=devicemapper &

    $ docker info

    Containers: 0
    Images: 0
    Storage Driver: devicemapper
     Pool Name: docker-252:0-147544-pool
     Pool Blocksize: 65.54 kB
     Backing Filesystem: extfs
     Data file: /dev/loop0
     Metadata file: /dev/loop1
     Data Space Used: 1.821 GB
     Data Space Total: 107.4 GB
     Data Space Available: 3.174 GB
     Metadata Space Used: 1.479 MB
     Metadata Space Total: 2.147 GB
     Metadata Space Available: 2.146 GB
     Thin Pool Minimum Free Space: 10.74 GB
     Udev Sync Supported: true
     Deferred Removal Enabled: false
     Data loop file: /var/lib/docker/devicemapper/devicemapper/data
     Metadata loop file: /var/lib/docker/devicemapper/devicemapper/metadata
     Library Version: 1.02.90 (2014-09-01)
    Execution Driver: native-0.2
    Logging Driver: json-file
    Kernel Version: 3.19.0-15-generic
    Operating System: Ubuntu 15.04
    <output truncated>

你选择的存储驱动会影响你的容器化应用的性能。所以理解和选择正确的存储驱动以及理解不同的存储驱动选项是很重要的。这篇文章的下面的部分，会给你选择存储驱动一些建议。

## 共享存储系统和Docker存储驱动

很多企业使用例如SAN或NAS阵列的共享存储系统。他们一般提供很好的性能和可用性，以及其他的比如精简配置，重复数据删除，压缩等高级特性。

Docker的存储驱动和数据卷同样可以在这些共享存储系统上使用。Docker可以使用这些系统提高性能和可用性。然而，Docker没有和这些后端的系统集成。

所有的Docker的存储驱动是基于Linux文件系统或者卷管理器的。请确保遵循了通过上层的存储驱动（文件系统或卷管理器）去操作共享存储系统的最佳实践。例如，如果你在*XYZ*的存储系统上使用ZFS存储驱动，确保遵循直接操作*XYZ*上层的ZFS文件系统的最佳实践。

## 你应该选择什么Docker存储驱动呢？

选择存储驱动需要考虑很多因素，但要记得这两点：

1. 不会有一个存储驱动适配于所有的场景
2. 存储驱动也都在持续的改进和发展

记得这两点后，以下的内容和表格会给你在选择存储驱动上一些指导。

### 稳定性
为了稳定和省事的Docker体验，你应该考虑下面这些：

- **使用你的发行版默认的存储驱动**. 当Docker安装的时候，它会基于你的系统配置选择默认的存储驱动，默认使用的存储驱动的主要考量点就是稳定性。更改默认的存储驱动可能会增加你遇到问题以及带来一些差别。

- **跟踪CS Engine默认指定的存储驱动
[CS Engine兼容性列表](https://success.docker.com/Help/Compatibility_Matrix)**. CS Engine是Docker Engine的商业支持版。它和开源的Docker Engine是同一个代码源的，但是它只有有限的支持的配置。那些*支持的配置*使用了最稳定和成熟的存储驱动。不使用这些配置可能会增加你遇到问题以及带来一些差别。

### 使用经验和专业知识

选择一个你的团队或者组织有经验的存储驱动。比如你如果经常使用RHEL或者它的下游的其他发行版，你可能早已对LVM和Device Mapper有经验了。所以你希望使用`devicemapper`存储驱动。

如果你并没有Docker提供的这些存储驱动的专业知识，然后你希望有一个稳定易用的Docker体验，你应该考虑使用Docker安装时默认使用的存储驱动。

### 考虑未来

很多人认为OverlayFS是Docker存储驱动的未来，然而它跟成熟的`aufs`或`devicemapper`相比不够稳定和成熟。所以你应该比使用更稳定的存储驱动更加小心的使用使用OverlayFS。

下面的图中列举了不同存储驱动的优缺点。当选择要使用的存储驱动是，通过上述的那些点来参考这个图表中的指导。

![](images/driver-pros-cons.png)

### Overlay vs Overlay2

OverlayFS基于同样的OverlayFS的存储技术实现了两个不同的存储驱动，他们有不同的实现方式，而且之间是不兼容的，切换这两种存储驱动需要重新创建所有的镜像内容。`overlay`驱动是原始的实现，并且在Docker 1.11和之前的版本只支持这个驱动。`overlay`的驱动有众所周知的一些限制包括inode的大量消耗和提交性能。`overlay2`主要解决了这些问题，但是只兼容与Linux Kernel 4.0和之后的版本。如果在4.0的kernel之前的版本，或者已经使用了`overlay`存储驱动，还是建议继续使用`overlay`的驱动。对于kernel的版本大于4.0并且之前也没用过`overlay`的存储驱动的用户，可以试一下`overlay2`的存储驱动。

> **Note**
> `overlay2` 的数据不会干扰 `overlay` 的数据. 然而，当切换成`overlay2`的存储驱动时，
> 用户最好删除之前的`overlay`的存储数据，避免数据冗余。

## 相关参考

* [理解镜像，容器，存储驱动](imagesandcontainers.md)
* [AUFS 存储驱动实践](aufs-driver.md)
* [Btrfs 存储驱动实践](btrfs-driver.md)
* [Device Mapper 存储驱动实践](device-mapper-driver.md)
