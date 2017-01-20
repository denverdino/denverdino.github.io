---
description: 学习如何优化Btrfs存储驱动.
keywords: 'container, storage, driver, Btrfs '
title: Docker和Btrfs实践
---

Btrfs是支持很多高级存储技术的支持copy-on-write的文件系统，所以很适配于Docker。Btrfs目前被包含在Linux kernel的主干中，并且它的on-disk-format（磁盘格式上）目前被认为是稳定的。然而，它的很多特性仍在开发中，用户应该考虑是否迁移到这个上面去。

Docker的`btrfs`的存储驱动在镜像和容器管理上使用了很多的Btrfs的特性。这些特性包括自动精简配置(thin provisioning)，copy-on-write，以及快照技术。

这篇文章中的Docker的Btrfs存储驱动会写成`btrfs`，以及Btrfs文件系统会写为Btrfs。

>**Note**: [商业支持版 Docker Engine (CS-Engine)](https://www.docker.com/compatibility-maintenance) 目前不支持`btrfs`存储驱动.

## Btrfs的未来

Btrfs长时间被誉为Linux文件系统的未来，在Linux Kernel主干代码的全面支持下，以及稳定的磁盘格式，以及针对稳定性的活跃的开发，目前它已经变成了现实。

就在Linux平台上的Docker而言，很多人认为`btrfs`将是潜在的`devicemapper`存储驱动的替代者。然而，目前来看`devicemapper`存储驱动被认为更安全，更稳定，以及更加*生产可用*。如果你目前理解并拥有Btrfs的经验的话，你应该在生产环境使用`btrfs`驱动。

## 使用Btrfs的镜像分层和共享

Docker使用Btrfs的 *subvolumes* 和 *snapshot* 管理磁盘上的镜像和容器的层的部分。Btrfs subvolumes看起来像是一个标准的Unix文件系统。比如他们可以有他们内部的目录结构，用于挂载广泛的Unix文件系统。

Subvolumes 原生支持copy-on-write技术，并且在在需要存储的空间时才会从下层的存储池中分配空间。并且Subvolume也支持嵌套和快照。下面的图中展示了4个subvolume。'Subvolume 2'和'Subvolume 3'是嵌套的，然后'Subvolume 4'展示了它内部的目录树结构。

![](images/btfs_subvolume.jpg)

Snapshot是一个时间点的对整个subvolume的读写拷贝。他们直接从属于他们创建自的subvolume。你可以像下图那样从snapshot创建snapshot。

![](images/btfs_snapshots.jpg)

Btrfs在需要时为subvolume何snapshot从底层的存储池中分配空间。分配的单位取决于一个*chunk*，并且一般*chunk* ~1GB大小。

Snapshot是Btrfs文件系统的一等公民。这一位这操作他们就像操作常规的subvolumes一样。创建Snapshot的技术是被直接集成在Btrfs文件系统中copy-on-write技术的设计实现。这意味着Btrfs snapshots高效的空间占用以及几乎没有的性能损失。下面这张图展示了subvolume和它的snapshot共享相同的数据。

![](images/btfs_pool.jpg)

Docker的`btffs`存储驱动通过Btrfs的subvolume或snapshot存储每个镜像的层和容器。一个镜像的基础层会被保存到subvolume，而下面的子层和容器被保存做snapshot。下图中展示了这个结构：

![](images/btfs_container_layer.jpg)

在Docker宿主机上通过`btrfs`驱动创建镜像和容器的流程如下：

1. 镜像的基础层存储在一个位于`/var/lib/docker/btrfs/subvolumes`下Btrfs的 *subvolume*中。
2. 后来的镜像的层作为福镜像的subvolume或者snapshot的一个 Btrfs *snapshot*。

	这个图中展示了一个三层的镜像。他的基础镜像是一个subvolume。第一层是一个对基础层subvolume的一个snapshot。第二层是第一层的snapshot的一个snapshot。

    ![](images/btfs_constructs.jpg)

Docker 1.10之后，镜像的的ID不再和`/var/lib/docker`目录名相同。

## 磁盘上的镜像和容器结构

镜像的层和容器在Docker宿主机文件系统都是可以在`/var/lib/docker/brtfs/subvolumes/`中看到的。然而，如前面介绍的，文件目录名并不是镜像的层ID。也就是说，就算容器停止后，也是存在容器目录的。这是因为`btrfs`存储驱动会在`/var/lib/docker/subvolumes`上挂载一个默认的，顶层的subvolume。所有的其他的subvolume和snapshot在这个subvolume下面，作为Btrfs的文件系统对象，而不是特殊的挂载。

因为Btrfs工作在文件系统级别而不是块存储级别，每个镜像和容器的层可以通过标准的Unix命令直接在文件系统中浏览到。例如下面的示例展示了一段在镜像的层中执行`ls -l`的输出：

    $ ls -l /var/lib/docker/btrfs/subvolumes/0a17decee4139b0de68478f149cc16346f5e711c5ae3bb969895f22dd6723751/

    total 0
    drwxr-xr-x 1 root root 1372 Oct  9 08:39 bin
    drwxr-xr-x 1 root root    0 Apr 10  2014 boot
    drwxr-xr-x 1 root root  882 Oct  9 08:38 dev
    drwxr-xr-x 1 root root 2040 Oct 12 17:27 etc
    drwxr-xr-x 1 root root    0 Apr 10  2014 home
    ...output truncated...

## Btrfs中容器的读写

容器是一个与镜像共享存储空间的的snapshot。snapshot的元数据中会有指向存储池中实际数据块的指针，这和subvolume中也是一样的。因此，对snapshot的读取实际上是对subvolume的读取，因此Btrfs驱动没有性能的损耗。

在容器中写新文件时，会触发按需分配(allocate-on-demand)的操作，实际会分配新的数据块给容器的snapshot。这个文件会写到一块新的空间上。这个按需分配(allocate-on-demand)的操作是Btrfs写操作原生支持的，写到subvolume中也是一样的。所以，写入在容器对应的snapshot写入新文件时也是原生的Btrfs的速度。

修改一个容器中的已有文件时会导致一个copy-on-write操作(*redirect-on-write*)。这个驱动会为snapshot分配新的空间并保留原有的数据。这个修改的数据会写入到新的空间中。然后驱动会更新文件系统的元数据中的指针指向新的数据。而原始的数据保留在原来的位置，以便于subvolumes和snapshots可能会扩展出新的树。这个行为也是copy-on-write类型的文件系统Btrfs原生支持的，只会有很小的性能损耗。

通过Btrfs，写或者更新大量的小文件会导致比较低性能，这点下面会介绍更多。

## 使用Btrfs使用Docker

`btrfs`存储驱动只能在将`/var/lib/docker`当做一个Btrfs文件系统挂载的Docker宿主机上使用。下面的流程展示了如何在Ubuntu 14.04 LTS上配置Btrfs。

### 准备工作

如果你早已经在宿主机上使用了Docker daemon并有自己的镜像，在尝试下面的步骤之前，先将他们`push`到Docker Hub上或者你自己的私有镜像仓库上。

停止Docker daemon。然后，确保你有例如`/dev/xvdb`的一个空闲块设备。这个设备的标识根据你的环境可能需要配置不同的值。

这个过程同样需要你的kernel加载合适的Btrfs的模块，可以用下面的命令验证：

    $ cat /proc/filesystems | grep btrfs

	        btrfs

### 在Ubuntu 14.04 LTS上配置Btrfs

假设你的系统已经满足了哪些前置要求，那么就跟着下面做：

1. 安装"btrfs-tools"软件包  

        $ sudo apt-get install btrfs-tools

        Reading package lists... Done
        Building dependency tree
        <output truncated>

2. 穿件Btrfs存储池

    Btrfs存储池可以通过`mkfs.btrfs`命令创建。可以通过`mkfs.btrfs`命令创建存储池的时候指定多个设备使存储池可以跨越多个设备。这里你通过单个的设备`/dev/xvdb`创建一个存储池。

        $ sudo mkfs.btrfs -f /dev/xvdb

        WARNING! - Btrfs v3.12 IS EXPERIMENTAL
        WARNING! - see http://btrfs.wiki.kernel.org before using

        Turning ON incompat feature 'extref': increased hardlink limit per file to 65536
        fs created label (null) on /dev/xvdb
            nodesize 16384 leafsize 16384 sectorsize 4096 size 4.00GiB
        Btrfs v3.12


    可以使用在你系统上挂载的别的设备替代`/dev/xvdb`。

    > **警告**:注意Btrfs的实验版本的警告。除非你对它有丰富的经验，Btrfs目前不推荐在生产部署使用。
    
3. 如果在Docker的本地存储区域中没有 `/var/lib/docker`目录，就创建这个目录.

        $ sudo mkdir /var/lib/docker

4. 配置系统配置，系统每次重启的时候自动挂载Btrfs文件系统。

    a. 获取 Btrfs 文件系统的 UUID.

        $ sudo blkid /dev/xvdb

        /dev/xvdb: UUID="a0ed851e-158b-4120-8416-c9b072c8cf47" UUID_SUB="c3927a64-4454-4eef-95c2-a7d44ac0cf27" TYPE="btrfs"

    b. 在`/etc/fstab`中创建一项用以在系统重启时自动挂载`/var/lib/docker`。记得通过上一条命令获得的UUID替换掉下面命令中的值保证下面的能够正常工作。

        /dev/xvdb /var/lib/docker btrfs defaults 0 0
        UUID="a0ed851e-158b-4120-8416-c9b072c8cf47" /var/lib/docker btrfs defaults 0 0

5. 挂载新的文件系统并验证。

        $ sudo mount -a

        $ mount

        /dev/xvda1 on / type ext4 (rw,discard)
        <output truncated>
        /dev/xvdb on /var/lib/docker type btrfs (rw)

    最后一行的输出表示`/dev/xvdb`通过`Btrfs`的方式挂载在`/var/lib/docker`。

现在你拥有了挂载在`/var/lib/docker`目录的Btrfs文件系统的挂载，docker daemon应该自动的通过`btrfs`存储驱动加载。

1. 启动Docker daemon。

        $ sudo service docker start

        docker start/running, process 2315

    启动Docker damon的方式取决于你在使用的Linux的发行版。

    你可以在`docker daemon`启动的地方增加`--storage-driver=btrfs`标志强制指定使用`btrfs`存储驱动。或者添加在Docker配置文件的`DOCKER_OPTS`中。

2. 通过`docker info`命令验证存储驱动。

        $ sudo docker info

        Containers: 0
        Images: 0
        Storage Driver: btrfs
        [...]

你的Docker宿主机现在就配置和使用了`btrfs`的存储驱动了。

## Btrfs 和 Docker 性能

在`btrfs`存储驱动下有下面这些因素影响Docker的性能。

- **页缓存**，Btrfs不支持页缓存，这意味着 *n* 个容器同事访问同一个文件需要 *n* 个缓存的实例。因此`btrfs`驱动对于Paas平台或者高密度容器部署不是好的选择。

- **小的写入**，容器中大量写入小文件或者Docker宿主机经常启停大量的容器会导致对Btrfs的chunks的使用。最终会导致对磁盘空间的过量使用的情况并导致无法工作。这是目前使用Btrfs的版本的主要的缺点。
    
    如果你使用`btrfs`存储驱动，要经常通过`btrfs filesys show`命令监控你的Btrfs文件系统的剩余空间。不要相信类似`df`这样的传统的Unix命令的输出；要使用Btrfs原生的命令。

- **连续写**，Btrfs通过日志结束往磁盘中写入数据。这种方式的性能可以高达一半的性能。

- **数据碎片**. 对于像Btrfs这样的copy-on-write的文件系统，自然会带来数据碎片的副作用，大量的随机写入更会加重这个问题，它在使用SSD上宿主机上会导致CPU的峰值，在使用旋转媒介的设备的上面会导致性能抖动。这两种都会导致比较低的性能。

    近期的Btrfs的版本允许你指定`autodefrag`作为mount时的参数。这个模式可以检测随机的读写，并整理他们。你在你的Docker宿主机上启用它时要事先做好测试。有一些测试表明这个参数会对Docker宿主机操作大量小文件或启停大量容器时有负面的性能影响。

- **固态硬盘(SSD)**. Btrfs对SSD存储有原生的优化，可以用过mount的时候指定`-o ssd`挂载参数打开打开这个优化。这些优化包括通过关闭SSDmedia上没有使用的*seek优化*的访问去提升SSD的写入性能。

    Btrfs同样支持磁盘的TRIM/Discard命令，然而如果挂载的时候指定`-o discard`参数可能会导致性能的问题。因此你需要在使用这个参数的时候自己先进行测试。

- **使用数据卷**. 数据卷提供最好的以及可预测的性能。这是因为它绕过存储驱动，不会增加由精简配置和copy-on-write带来的额外的性能消耗。由于这个原因，你应该将更重负载的数据读写放到数据卷中。

## 相关信息

* [理解镜像，容器和存储驱动](imagesandcontainers.md)
* [选择存储驱动](selectadriver.md)
* [AUFS 存储驱动实践](aufs-driver.md)
* [Device Mapper 存储驱动实践](device-mapper-driver.md)
