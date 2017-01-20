---
description: 如何使用Device mapper存储驱动管理Docker
keywords: container, storage, driver, device mapper
title: Docker 与 Device Mapper 存储驱动
---

Device Mapper是一个基于内核的存储框架，包含Linux上很多卷管理的技术。Docker的`devicemapper`存储驱动使用了这个框架中的thin provisioning和snapshotting的能力做镜像和容器的管理。这篇文章中`devicemapper`指Device Mapper存储驱动，而`Device Mapper`指内核存储框架。

>**Note**: [RHEL and CentOS Linux上的商业支持版的Docker Engine (CS-Engine)](https://www.docker.com/compatibility-maintenance) 需要你使用`devicemapper` 存储驱动.

## AUFS的替代者

Docker之前只能在Ubuntu或者Debian Linux上使用AUFS作为它的存储的后端。Docker变得流行起来后，很多公司希望在Red Hat Enterprise Linux (RHEL)上使用Docker。不行的是因为Linux kernel并没有合并AUFS，RHEL不能使用AUFS。

为了改变这个情况，Red Hat的开发者研究如何将AUFS合并到kernel的主管。最终，他们决定通过另外一种方式，即开发一种新的存储后端。并且他们基于现有的`Device Mapper`技术去开发新的存储后端。

Red Hat通Docker公司合作，贡献了这个新的驱动。并且作为合作的结果，Docker Engine也重新设计支持可插拔的不同存储后端。所以`devicemapper`变成了Docker支持的第二个存储驱动。

Device Mapper从2.6.9的Linux内核就被包含了进来。它是RHEL的Linux发型版家族的重要的组成部分。这意味着`devicemapper`存储驱动是基于稳定的代码构建的，并且有真实的生产环境产品在使用，以及强大的社区支持。


## 镜像的分层和共享

`devicemapper`驱动在它的虚拟设备中存储每个镜像和容器。这些设备是 thin-provisioned copy-on-write的快照设备。Device Mapper技术工作在块设备级别而不是文件级别。这意味着`devicemapper`存储驱动的thin provisioning操作和copy-on-write操作工作在磁盘块级别，而不是整个文件。

>**Note**: Snapshots 也被成为 *thin devices* 或者 *virtual devices*. 
> 在`devicemapper`存储驱动中他们表示同样的意思。

`devicemapper`创建镜像的处理流程如下：

1. `devicemapper` 创建一个 thin pool.

	这个 pool是从块设备创建的或者通过稀疏文件的loop挂载创建的（下面会详细介绍）

2. 然后创建一个 *基础设备*.

	一个基础设备是一个thin设备加上文件系统，你可以通过`docker info`命令中的`Backing filesystem`值看到这个具体的文件系统。

3. 每个新的镜像和镜像的层作为这个基础设备的一个快照。

	他们都是thin provisioned出来的copy-on-write的快照，这一意味着他们初始的时候是空的，只有在数据写入到其中的时候才会真正的占用空间。

通过`devicemapper`，容器层就是创建它的镜像的snapshot。和镜像一样，容器的snapshot也是provisioned copy-on-write的snapshot。容器的snapshot中保存了所有容器中的改变。`devicemapper`只会在数据写入到容器的时候才会从pool中实际的分配空间。

下面这个总览图展示了基础的设备以及两个镜像。

![](images/base_device.jpg)

如果你仔细观察，你会发现他的快照是一路向下的，每个镜像的层是它下层的层的一个快照。而最下面的层是pool中的基础设备的快照。这个基础设备是`Device Mapper`的组件，不是Docker镜像的层。

一个容器是创建它的镜像的快照。下面这张图展示了两个基于Ubuntu镜像和Busybox镜像的容器。

![](images/two_dm_container.jpg)


## 使用devicemapper读取

让我们一起看下在使用`devicemapper`存储驱动读取和写入时会发生什么。下面这张图展示了在一个示例容器中读取一个块(`0x44f`)。

![](images/dm_container.jpg)

1. 应用程序在容器中对`0x44f`发起读取的请求。

	因为容器是镜像的一个快照，所以它并没有数据，不过它有一个指针(PTR)指向真正数据存储的镜像层对应的snapshot。

2. 存储驱动跟踪指针到`a005...`的镜像中的`0xf33`块。

3. `devicemapper`存储驱动拷贝`0xf33`的镜像snapshot中的块内容到容器的内存中。

4. 存储驱动返回数据给需要的应用程序。

## 写入数据的例子

使用`devicemapper`驱动，往容器中写入数据时，依赖于它的 *allocate-on-demand*操作。使用copy-on-write操作。因为Device Mapper是基于块的技术，所以这些操作发生在块的层次。

例如，当对一个大文件做很小的改动时，`devicemapper`存储驱动并不会拷贝整个文件。它仅仅拷贝修改的块，每个块的大小是64KB。

### 写入新数据

如果写入56KB的新数据到一个容器中：

1. 应用程序在容器中发起了一个写入56KB的写入新数据的请求。

2. allocte-on-demand操作会分配一块新的64KB的块给容器的snapshot。

	如果写入的数据大于64KB，会分配给容器的snapshot多个块。

3. 数据写入到新分配的块中。

### 复写已存在的数据

第一次修改已存在的数据时。

1. 应用程序发起修改某些数据的请求。

2. copy-on-write操作找到需要更新的块。

3. 这个操作会分配新的空的块给容器的snapshot，并拷贝数据到这些块中。

4. 修改的数据将写入到新分配的块中。

容器中的应用程序无需感知这些allocate-on-demand和copy-on-write操作。不过他们可能会增加应用程序读写操作的时间。

## 配置Docker使用`device mapper`

The `devicemapper` is the default Docker storage driver on some Linux
distributions. This includes RHEL and most of its forks. Currently, the
following distributions support the driver:

* RHEL/CentOS/Fedora
* Ubuntu 12.04
* Ubuntu 14.04
* Debian

Docker hosts running the `devicemapper` storage driver default to a
configuration mode known as `loop-lvm`. This mode uses sparse files to build
the thin pool used by image and container snapshots. The mode is designed to
work out-of-the-box with no additional configuration. However, production
deployments should not run under `loop-lvm` mode.

You can detect the mode by viewing the `docker info` command:

```bash
$ sudo docker info

Containers: 0
Images: 0
Storage Driver: devicemapper
 Pool Name: docker-202:2-25220302-pool
 Pool Blocksize: 65.54 kB
 Backing Filesystem: xfs
 [...]
 Data loop file: /var/lib/docker/devicemapper/devicemapper/data
 Metadata loop file: /var/lib/docker/devicemapper/devicemapper/metadata
 Library Version: 1.02.93-RHEL7 (2015-01-28)
 [...]
 ```

The output above shows a Docker host running with the `devicemapper` storage
driver operating in `loop-lvm` mode. This is indicated by the fact that the
`Data loop file` and a `Metadata loop file` are on files under
`/var/lib/docker/devicemapper/devicemapper`. These are loopback mounted sparse
files.

### Configure direct-lvm mode for production

在生产环境推荐使用`direct-lvm`模式配置。这个模式直接使用块设备创建thin pool。下面的流程展示如何使用`direct-lvm`配置Docker宿主机使用`devicemapper`驱动。

> **警告:** 如果你之前已经在你的宿主机上使用了Docker并且有需要保存的镜像，请先将他们Push到
> Docker Hub或者你私有的Docker Trusty Registry。

下面的流程将创建一个逻辑卷，并将它配置为一个thin poll作为存储池的后端存储。它需要你事先拥有一个足够空间的块设备在`/dev/xvdf`上。这个设备的标识和卷的大小根据你的环境会有所不同，你需要在这个过程中做适配和调整。这个流程也需要Docker daemon是停止状态。

1. 登录到你需要配置的Docker宿主机，并停止Docker daemon。

2. 安装LVM2的软件包
	LVM2的软件包包括管理逻辑卷的用户态的工具集

3. 通过你的块设备创建一个物理卷，并放置在`/dev/xvdf`。

	```bash
	$ pvcreate /dev/xvdf
	```

4. 创建一个叫'docker'名的group

	```bash
	$ vgcreate docker /dev/xvdf
	```

5. 创建一个名为`thinpool`的pool

	在这个例子中，逻辑数据是'docker'卷组数据的95%，留出这些空闲空间是为了在剩余数据或者元数据空间不足到stopgap时自动扩容。

	```bash
	$ lvcreate --wipesignatures y -n thinpool docker -l 95%VG
	$ lvcreate --wipesignatures y -n thinpoolmeta docker -l 1%VG
	```

6. 将poll转换成一个thin poll

	```bash
	$ lvconvert -y --zero n -c 512K --thinpool docker/thinpool --poolmetadata docker/thinpoolmeta
	```

7. 通过`lvm`配置文件配置thin pool的自动扩展

	```bash
	$ vi /etc/lvm/profile/docker-thinpool.profile
	```

8. 配置 'thin_pool_autoextend_threshold' 值.

	这个值是上一步中配置`lvm`中自动扩展的空间，是空间的百分比（100 = 关闭）

	```
	thin_pool_autoextend_threshold = 80
	```

9. 当thin pool自动扩展发生时修改 `thin_pool_autoextend_percent` 的值。
	
	这个值修改了增加thin pool的百分比(100 = 关闭)。

	```
	thin_pool_autoextend_percent = 20
	```

10. 检查你的工作，你的`docker-thinpool.profile`文件应该与下面类似：

	`/etc/lvm/profile/docker-thinpool.profile`示例文件:

	```
	activation {
		thin_pool_autoextend_threshold=80
		thin_pool_autoextend_percent=20
	}
	```

11. 使你的新的lvm的配置生效

	```bash
	$ lvchange --metadataprofile docker-thinpool docker/thinpool
	```

12. 验证 `lv` 是被监控的.

	```bash
	$ lvs -o+seg_monitor
	```

13. 如果之前Docker dameon启动过，先移除现有的存储驱动的目录。

    迁移存储驱动意味着移动你的Docker的所有的镜像，容器和volume。这些命令将移动之前的`/var/lib/docker`目录到一个新的`/var/lib/docker.bk`目录。如果下面的步骤失败了，你需要加载之前的内容时，你可以将`/var/lib/docker`替换成`/var/lob/docker.bk`目录的内容。
    
    ```bash
    $ mkdir /var/lib/docker.bk
    $ mv /var/lib/docker/* /var/lib/docker.bk
    ```

14. 通过指定`devicemapper`特定的参数配置`devicemapper`驱动

    现在你的存储已经配置好了，再配置Docker daemon使用这个存储。下面有两种方式可以做到，你可以设置启动daemon的命令行的参数：

    ```bash
    --storage-driver=devicemapper --storage-opt=dm.thinpooldev=/dev/mapper/docker-thinpool --storage-opt=dm.use_deferred_removal=true --storage-opt=dm.use_deferred_deletion=true
    ```

	你也可以在`daemon.json`中设置启动的配置，例如：

    ```json
    {
      "storage-driver": "devicemapper",
       "storage-opts": [
         "dm.thinpooldev=/dev/mapper/docker-thinpool",
         "dm.use_deferred_removal=true",
         "dm.use_deferred_deletion=true"
       ]
    }
    ```

    >**Note**: 同时设置 `dm.use_deferred_removal=true` and `dm.use_deferred_deletion=true` 已避免挂载点泄露。

15. 如果通过systemd的unit或drop-in文件修改daemon的配置，需要reload systemd以便扫描到这些修改。

	```bash
	$ systemctl daemon-reload
	```

16. 启动Docker daemon.

	```bash
	$ systemctl start docker
	```

当你启动了Docker daemon后，要保证监控你的thin poll和卷组的空闲空间。当卷组扩容之后，它可能会用尽空间。使用`lvs`或者`lvs -a`可以看到数据和元数据的空间。如果要兼容卷组的空闲空间，可以使用`vgs`命令。

当thin poll触及到它的阈值时，可以通过日志看到它的自动扩容，可以通过下面的命令查看到日志：

```bash
$ journalctl -fu dm-event.service
```

当你验证完配置是正确的了，你就可以删除`/var/lib/docker.bk`包含之前配置的目录了。

```bash
$ rm -rf /var/lib/docker.bk
```

如果你在使用thin pool的时候反复遇到问题，你可以使用`dm.min_free_space`参数调优Engine的行为。这个参数保证在空闲空间小时写入数据失败，并返回报警。更多信息可以参考<a
href="/../../reference/commandline/dockerd/#storage-driver-options"
target="_blank">Docker Engine参考之存储驱动配置</a>。


### 在宿主机上检测devicemapper的结构

你可以使用`lsblk`命令看到`devicemapper`存储驱动在`poll`上面创建的设备文件。

```bash
$ sudo lsblk
NAME			   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
xvda			   202:0	0	 8G  0 disk
└─xvda1			   202:1	0	 8G  0 part /
xvdf			   202:80	0	10G  0 disk
├─vg--docker-data		   253:0	0	90G  0 lvm
│ └─docker-202:1-1032-pool 253:2	0	10G  0 dm
└─vg--docker-metadata	   253:1	0	 4G  0 lvm
  └─docker-202:1-1032-pool 253:2	0	10G  0 dm
```

这个图中展示了上面的`lsblk`命令的设备的结构。

![](images/lsblk-diagram.jpg)

在图中，包含之前创建的`数据`和`元数据`的名为`Docker-202:1-1032-poll`的池子。`devicemapper`的驱动会设置pool的名字如下：

```
Docker-MAJ:MIN-INO-pool
```

`MAJ`, `MIN` and `INO` 指最大和最小的设备号以及inode。

因为Device Mapper在块级别工作，所以它查看镜像的层及容器的层的区别比较复杂。Docker 1.10和之后的版本在`/var/lib/docker`不在使用镜像层的ID作为目录名。不过，这里仍有两个关键的目录，`/var/lib/docker/devicemapper/mnt`目录包含了容器和镜像的挂载点。`/var/lib/docker/devicemapper/metadata`目录包含了每个镜像的层或容器的snapshot。这个文件通过JSON格式包含层的元信息。

## 提升一个运行中的设备的大小

你可以实时的增加一个运行中的thin-pool设备的大小。这个功能在逻辑卷或者卷组满了的时候很有用。

### loop-lvm 配置

在这个场景中，thin pool配置成使用`loop-lvm`的模式。下面展示了已经配置好的输出的`docker info`信息：

```bash
$ sudo docker info

Containers: 0
 Running: 0
 Paused: 0
 Stopped: 0
Images: 2
Server Version: 1.11.0
Storage Driver: devicemapper
 Pool Name: docker-8:1-123141-pool
 Pool Blocksize: 65.54 kB
 Base Device Size: 10.74 GB
 Backing Filesystem: ext4
 Data file: /dev/loop0
 Metadata file: /dev/loop1
 Data Space Used: 1.202 GB
 Data Space Total: 107.4 GB
 Data Space Available: 4.506 GB
 Metadata Space Used: 1.729 MB
 Metadata Space Total: 2.147 GB
 Metadata Space Available: 2.146 GB
 Udev Sync Supported: true
 Deferred Removal Enabled: false
 Deferred Deletion Enabled: false
 Deferred Deleted Device Count: 0
 Data loop file: /var/lib/docker/devicemapper/devicemapper/data
 WARNING: Usage of loopback devices is strongly discouraged for production use. Use `--storage-opt dm.thinpooldev` to specify a custom block storage device.
 Metadata loop file: /var/lib/docker/devicemapper/devicemapper/metadata
 Library Version: 1.02.90 (2014-09-01)
Logging Driver: json-file
[...]
```

`Data Space`数值展示了这个pool总共有100GB大小。这个例子扩展这个pool到200GB。

1. 罗列设备的大小。

	```bash
	$ sudo ls -lh /var/lib/docker/devicemapper/devicemapper/

	total 1175492
	-rw------- 1 root root 100G Mar 30 05:22 data
	-rw------- 1 root root 2.0G Mar 31 11:17 metadata
	```

2. 扩展`data`文件到`metadata`的大小（大约 200GB)。

	```bash
	$ sudo truncate -s 214748364800 /var/lib/docker/devicemapper/devicemapper/data
	```

3. 验证文件大小被改变了.

	```bash
	$ sudo ls -lh /var/lib/docker/devicemapper/devicemapper/

	total 1.2G
	-rw------- 1 root root 200G Apr 14 08:47 data
	-rw------- 1 root root 2.0G Apr 19 13:27 metadata
	```

4. 重载数据的loop设备

	```bash
	$ sudo blockdev --getsize64 /dev/loop0

	107374182400

	$ sudo losetup -c /dev/loop0

	$ sudo blockdev --getsize64 /dev/loop0

	214748364800
	```

5. 重载devicemapper的thin pool.

	a. 首先得到pool的名字.

	```bash
	$ sudo dmsetup status | grep pool

	docker-8:1-123141-pool: 0 209715200 thin-pool 91
	422/524288 18338/1638400 - rw discard_passdown queue_if_no_space -
	```

	名字是冒号前面的字符串。

	b. 首先展示device mapper的table。

	```bash
	$ sudo dmsetup table docker-8:1-123141-pool

	0 209715200 thin-pool 7:1 7:0 128 32768 1 skip_block_zeroing
	```

	c. 计算现在的thin的全部的扇区数。

	修改表信息中的第二个数值到映射到512个字节的扇区个数。例如新的loop的大小是200GB，那么久修改第二个数字到419430400。

	d. 使用新的扇区配置重载thin pool。

	```bash
	$ sudo dmsetup suspend docker-8:1-123141-pool \
	    && sudo dmsetup reload docker-8:1-123141-pool --table '0 419430400 thin-pool 7:1 7:0 128 32768 1 skip_block_zeroing' \
	    && sudo dmsetup resume docker-8:1-123141-pool
	```

#### device_tool

Docker项目的`contrib`目录的核心发布不包含这个。这些工具很有用但是可能会过时。<a
href="https://goo.gl/wNfDTi">在这个目录下的`device_tool.go`</a>可以帮助你修改loop-lvm的thin pool大小。

如果需要使用这个工具，首先编译它，然后使用下面的命令修改pool的大小:

```bash
$ ./device_tool resize 200GB
```

### direct-lvm模式配置

在这个例子中，你可以扩展运行中的使用`direct-lvm`配置的设备的大小。下面的例子使用的是`/dev/sdh1`磁盘分区。

1. 扩展卷组 (VG) `vg-docker`.

	```bash
	$ sudo vgextend vg-docker /dev/sdh1

	Volume group "vg-docker" successfully extended
	```

	你的卷组可能是别的名字.

2. 扩展`vg-docker/data`对应的 `data` 逻辑卷(LV) 

	```bash
	$ sudo lvextend  -l+100%FREE -n vg-docker/data

	Extending logical volume data to 200 GiB
	Logical volume data successfully resized
	```

3. 重载devicemapper的thin pool。

	a. 获得pool的名字。

	```bash
	$ sudo dmsetup status | grep pool

	docker-253:17-1835016-pool: 0 96460800 thin-pool 51593 6270/1048576 701943/753600 - rw no_discard_passdown queue_if_no_space
	```

	冒号之前的字符串就是pool的名字。

	b. 打印出device mapper的设备表。

	```bash
	$ sudo dmsetup table docker-253:17-1835016-pool

	0 96460800 thin-pool 252:0 252:1 128 32768 1 skip_block_zeroing
	```

	c. 计算thin pool的正在使用的总的扇区数，我们可以通过`blockdev`得到目前的数据的卷的大小。
	
	修改表信息中的第二个数值到映射到512个字节的扇区个数。例如新的数据卷的大小是`264132100096`，那么修改第二个数值到`515883008`。

	```bash
	$ sudo blockdev --getsize64 /dev/vg-docker/data

	264132100096
	```

	d. 使用新的卷的数值重载thin pool使用新的大小。

	```bash
	$ sudo dmsetup suspend docker-253:17-1835016-pool \
	    && sudo dmsetup reload docker-253:17-1835016-pool --table  '0 515883008 thin-pool 252:0 252:1 128 32768 1 skip_block_zeroing' \
	    && sudo dmsetup resume docker-253:17-1835016-pool
	```

## Device Mapper 和 Docker 性能

理解 allocate-on-demand和 copy-on-write操作对容器性能的影响是很重要的。

### Allocate-on-demand 性能影响

`devicemapper`存储驱动通过allocate-on-demand操作分配给容器新的块。这意味容器中的应用写入新的内容的时候，一个或者多个块会被从pool中分配并映射到容器中。

所有的块的大小都是64KB。小于64KB的写入同样会导致分配64KB的块。写入超过64KB的数据会分配多个块。这个会影响到容器的性能，特别是容器中昌盛很多小的写入时。不过一旦一个block分配给容器后，后续的读取和写入会直接访问到这个块。

### Copy-on-write 性能影响

容器中第一次修改容器中已存在的数据时，`devicemapper`存储驱动会做一个copy-on-write操作。它会从镜像的snapshot中拷贝数据到容器的snapshot中。这个会对容器的性能有显著的影响。

所有的copy-on-write操作同样拥有64KB的粒度。修改一个1GB的文件的32KB会导致复制一个64KB的块到容器的snapshot中。这个操作会比需要拷贝整个1GB文件的文件级别的copy-on-write操作要有显著的性能优势。

在实践中发现，容器产生大量小的块(<64KB)的写入的话，`devicemapper`性能不如AUFS。

### 其他的 device mapper 性能的考虑

下面是别的一些影响`devicemapper`存储驱动的东西。

- **模式** 默认的Docker运行`devicemapper`存储驱动的模式是`loop-lvm`的方式，这个模式使用稀疏文件，会导致比较差的性能。**不推荐在生产环境使用**。在生产环境推荐使用`direct-lv`的方式，也就是说存储驱动直接写入到原始块设备。

- **高速存储** 如果希望有更好的性能，你需要将`Data file`和`Metadata file`存储在高速存储上，比如SSD。他们可以直接挂载存储，或者通过SAN或NAS阵列的方式挂载。

- **内存使用** `devicemapper`在Docker存储驱动中内存效率不占优。容器同时加载同一个文件会在内存中有多个拷贝实例。这个会对你的Docker宿主机的内存有影响。所以`devicemapper`存储驱动并不是Paas和高密度的使用场景的最佳的选择。

最后一点, 数据卷提供最好的以及可预测的性能。这是因为它绕过存储驱动，不会增加由精简配置和copy-on-write带来的额外的性能消耗。由于这个原因，你应该将更重负载的数据读写放到数据卷中。

## 相关信息

* [理解镜像，容器，存储驱动](imagesandcontainers.md)
* [选择存储驱动](selectadriver.md)
* [AUFS 存储驱动实践](aufs-driver.md)
* [Btrfs 存储驱动实践](btrfs-driver.md)
* [daemon 配置参考](../../reference/commandline/dockerd.md#storage-driver-options)
