---
description: Learn how to optimize your use of AUFS driver.
keywords: 'container, storage, driver, AUFS '
title: Docker and AUFS in practice
---

AUFS是Docker支持的第一个存储驱动程序。
因此，它与Docker有着悠久而密切的历史，非常稳定，很多应用于真实的生产环境的例子，并且拥有强大的社区支持。
AUFS有以下几个功能，使其成为Docker存储的不错选择。 这些功能包括：

- 容器启动快速。
- 高效利用存储。
- 高效利用内存。

尽管它很强大、与Docker有悠久的历史，但是一些Linux发行版并不支持AUFS。
这通常是因为在较早版本的一些Linux内核中没有AUFS相关的支持。

以下章节，介绍了一些AUFS的功能以及它们如何与Docker配合使用。

## 镜像分层和与AUFS共享

AUFS是一个 *联合文件系统* 。这意味着它在单个Linux主机上使用多个目录，将它们堆叠在彼此之上，并提供单个统一视图。
为了实现这一点，AUFS使用了 *union mount* 。

AUFS堆叠多个目录，并通过单个挂载点将其显示为统一视图。
堆栈中的所有目录以及挂载点必须都存在于同一Linux主机上。AUFS引用它作为*分支*堆叠的每个目录。

在Docker中，AUFS联合挂载支持镜像分层。AUFS存储驱动程序使用此联合文件系统来实现Docker镜像分层。
AUFS分支对应于Docker镜像层。下图显示了基于`ubuntu:latest`镜像的Docker容器。

![](images/aufs_layers.jpg)

此图显示每个镜像层和容器层在Docker主机中被存储为`/var/lib/docker/`下的目录。
联合挂载点提供所有镜像的统一视图。同时从Docker 1.10开始，镜像层ID不再使用存储其数据的目录的名称。

AUFS还支持写时复制（CoW）。但是不是所有存储驱动程序都支持。

## 使用AUFS进行容器读写

Docker利用AUFS写时复制技术，允许映像共享并最小化磁盘空间的使用。
AUFS在文件层面进行工作。这意味着所有AUFS写时复制操作都复制整个文件，即使只修改了文件的一小部分。
此特性可以对容器性能产生显着影响，特别是如果要复制的文件较大，有大量的镜像层，或者写时复制操作必须搜索一个层次复杂的目录树。

例如，考虑在容器中运行的应用程序需要向大键值存储中添加单个新值。
如果这是第一次修改文件，它在容器的最高可写层中将不存在。
因此，写时复制必须 *复制并向上转移* 文件。AUFS存储驱动程序搜索每个镜像层的文件。
搜索顺序是从上到下。当找到这个文件时，整个文件被复制到容器的最高可写层。在最高层这个文件可以被打开和修改。

## 在AUFS存储中删除一个文件

AUFS存储驱动程序通过将 *空占位文件* 放在容器的顶层中，来完成删除文件。
空占位文件有效地掩盖了下面的只读映像层中存在的文件。下面的简化图显示了一个具有三个镜像层的镜像容器。

![](images/aufs_delete.jpg)

`file3`从容器中删除。因此，AUFS存储驱动程序在容器的顶层中放置一个空占位文件。
这个空占位文件有效的“删除”了存在于下面只读层的文件，从而有效地从容器中“删除”了`file3`。
即使文件真实存在于只读层中。

## 使用AUFS存储驱动程序重命名目录

AUFS不完全支持为目录调用`rename(2)`。
即使源和目标路径都在同一AUFS层上，除非目录没有子节点，否则它将返回`EXDEV`（“不允许跨设备链接”）。

因此，您的应用程序必须设计为支持处理`EXDEV`并回退到“复制和解除链接”的策略。

## 为Docker配置AUFS

您只能在安装了AUFS的Linux系统上使用AUFS存储驱动程序。
使用以下命令可以用于确定系统是否支持AUFS。

    $ grep aufs /proc/filesystems

    nodev   aufs

这个输出表示系统支持AUFS。一旦您验证了系统支持AUFS，您必须让Docker Daemon使用它。
你可以从命令行使用`dockerd`命令：

    $ sudo dockerd --storage-driver=aufs &

或者，您可以将`--storage-driver=aufs`选项添加到Docker配置文件的`DOCKER_OPTS`那一行。

    # Use DOCKER_OPTS to modify the daemon startup options.
    DOCKER_OPTS="--storage-driver=aufs"

一旦Docker Daemon运行，请使用`docker info`命令验证存储驱动程序。

    $ sudo docker info

    Containers: 1
    Images: 4
    Storage Driver: aufs
     Root Dir: /var/lib/docker/aufs
     Backing Filesystem: extfs
     Dirs: 6
     Dirperm1 Supported: false
    Execution Driver: native-0.2
    ...output truncated...

上面的输出显示Docker Daemon正在`ext4`文件系统之上运行AUFS存储驱动程序。

## 本地存储与AUFS

由于`dockerd`使用AUFS驱动程序运行，
驱动程序会在Docker宿主机的`/var/lib/docker/aufs/`下存储镜像和容器。

### 镜像

镜像层及其内容存储在`/var/lib/docker/aufs/diff/`下。
对于Docker 1.10和更高版本，镜像层ID不与目录名称相对应。

`/var/lib/docker/aufs/layers/`目录中包含有关如何堆叠镜像层的元数据。
此目录包含Docker主机上每个镜像或容器层的一个文件（尽管文件名不再与镜像层ID匹配）。
每个文件内部是在堆叠镜像中存在于其下的目录名称。

    $ cat /var/lib/docker/aufs/layers/91e54dfb11794fad694460162bf0cb0a4fa710cfa3f60979c177d920813e267c

    d74508fb6632491cea586a1fd7d748dfc5274cd6fdfedee309ecdcbc2bf5cb82
    c22013c8472965aa5b62559f2b540cd440716ef149756e7b958a1b2aba421e87
    d3a1f33e8a5a513092f01bb7eb1c2abf4d711e5105390a3fe1ae2248cfde1391

在镜像中的基本镜像层下面没有镜像层，因此其文件为空。

### 容器

运行容器挂载在`/var/lib/docker/aufs/mnt/<container-id>`下。
这是将存在容器和所有底层镜像层作为单个统一视图的AUFS联合挂载点的位置。
如果容器没有运行，它仍然有一个目录，但它是空的。这是因为AUFS只在其运行时装载容器。
对于Docker 1.10和更高版本，容器ID不再对应于`/var/lib/docker/aufs/mnt/<container-id>`下的目录名称。

容器元数据和正在运行的容器中的各种配置文件存储在`/var/lib/docker/containers/<container-id>`中。
此目录中的文件对于系统上的所有容器都存在，包括停止的容器。
当容器正在运行时，容器的日志文件也在此目录中。

容器的可写层存储在`/var/lib/docker/aufs/diff/`下的目录中。
对于Docker 1.10和更高版本，容器ID不再对应于目录名称。但是，容器的可写层仍然存在，并且作为AUFS存储的顶部可写层堆叠，并且存储对容器的所有改变。
即使容器已停止，目录也存在。这意味着重新启动容器不会丢失对它所做的更改。一旦容器被删除，这个目录中的可写层被删除。

## AUFS和Docker的性能表现

总结性能相关方面提到的一些问题：

- AUFS存储驱动程序是PaaS和其他类似场景的不错选择，其中容器密度很重要。
这是因为AUFS能够在多个正在运行的容器之间高效地共享镜像，从而实现容器的快速启动和最小化磁盘空间的使用。

- AUFS在镜像层和容器之间共享文件的基本机制，使得其使用系统页缓存非常有效。

- AUFS存储驱动程序可能给容器写入操作中带来显着的延迟。
这是因为第一次容器写入任何文件，这个文件都必须被定位和复制到容器的最高可写层。
当这些文件存在于许多镜像层之下并且文件本身很大时，这将带来延迟和很高的复杂性。

最后一点。数据卷提供最佳和最可预期的性能。
因为它们绕过了存储驱动程序，并且不会因为精简配置和写时复制而引入的任何潜在的开销。
因为这些原因，您可能会希望让数据卷承担比较大的负载。

## AUFS的兼容性

总结AUFS与其他文件系统不兼容的方面：

- AUFS不完全支持`rename(2)`系统调用。您的应用程序需要检测其`EXDEV`通知，并回退到“复制和解除链接”的策略。

## 相关参考

* [Understand images, containers, and storage drivers](imagesandcontainers.md)
* [Select a storage driver](selectadriver.md)
* [Btrfs storage driver in practice](btrfs-driver.md)
* [Device Mapper storage driver in practice](device-mapper-driver.md)
