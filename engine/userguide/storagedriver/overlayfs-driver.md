---
description: 学习使用OverlayFS 存储驱动优化Docker。
keywords: container, storage, driver, OverlayFS
title: Docker和OverlayFS实践
---

OverlayFS 是一个类似于AUFS的 *联合文件系统*。与AUFS对比的话，Overlay：

* 更简单的设计
* 从3.18的内核开始就合并到Linux主线上
* 速度更快

因此，在Docker社区中OverlayFS正在快速的流行起来，并且看起来将是AUFS的继承者。作为未来的前景，它目前还不算成熟。因此在生产环境使用它时要小心谨慎。

Docker的`overlay`存储驱动利用了很多OverlayFS的特性来管理磁盘上镜像和容器的数据结构。

从1.12的版本之后，Docker也提供了比`overlay`在inode使用上更高效的`overlay2`的存储驱动。`overlay2`的存储驱动只兼容与Linux kernel 4.0和之后的版本。

关于`overlay`和`overlay2`的对比，可以参考[选择存储驱动](selectadriver.md#overlay-vs-overlay2)。

>**Note**: 在合并到kernel主干开始，OverlayFS的*kernel模块*就从`overlayfs`改名成了
> `overlay`。因此你可能会看到在某些文档中有这这种不同的术语。不过这篇文档中使用“OverlayFS”指
> 代Overlay文件系统，`overlay`/`overlay2`指代Docker的存储驱动。

## 通过OverlayFS (`overlay`) 的镜像分层和共享

OverlayFS 在Linux宿主机上使用一层在另外一层上面的两个目录组成一个标准的视图。这些目录京城被称为 *层* 并且这种技术被称为*联合挂载*。OverlayFS使用一个"lowerdir"作为下层，和"upperdir"作为上层。而组合成的标准视图称之为"merged"。

下面的图中展示了Docker镜像和容器是如何分层的。镜像的层是作为"lowerdir"，容器的层作为"upperdir"。而通过称为"merged"的一个目录作为统一的视图，用作容器的挂载点。这个图中展示了Docker使用OverlayFS的组成结构。

![](images/overlay_constructs.jpg)

在镜像的层和容器的层中可以包含相同的文件。当这样的事件发生时，容器层("upperdir")的文件会优先显示出来，并忽略下层的镜像层("lowerdir")中的文件。容器的挂载("merged")展示了统一的视图。

`overlay`驱动仅工作于两层。这就意味着多层的镜像不能通过多层的OverlayFS层来实现。所以，镜像的每一层都会在`/var/lib/docker/overlay`下面有他自己的文件夹。然后使用硬链接的方式指向下层中的共享的数据。Docker 1.10之后，镜像的层ID不再和`/var/lib/docker`的镜像ID绑定。

在创建一个容器时，`overlay`的驱动镜像组合最上层对应的目录和一个新的目录作为容器的层。镜像的最上层的目录作为overlay的"lowerdir"并且是只读的。而新创建的目录是容器的"upperdir"，是可写的。

### 示例: 镜像和容器在磁盘上的结构(`overlay`)

下面的在Docker宿主机的`docker pull`的命令展示了下载一个包含5层的一个docker镜像。

    $ sudo docker pull ubuntu

    Using default tag: latest
    latest: Pulling from library/ubuntu

    5ba4f30e5bea: Pull complete
    9d7d19c9dc56: Pull complete
    ac6ad7efd0f9: Pull complete
    e7491a747824: Pull complete
    a3ed95caeb02: Pull complete
    Digest: sha256:46fb5d001b88ad904c5c732b086b596b92cfb4a4840a3abd0e35dbb6870585e4
    Status: Downloaded newer image for ubuntu:latest

每一层镜像在`/var/lib/docker/overlay`下面有自己的目录。每个镜像的层将内容保存在这个这个目录中。

下面的命令展示了刚才pull到的镜像的保存层内容的5个目录。不过，如你所见，镜像的ID并不匹配于`/var/lib/docker/overlay`目录的目录名，这个在Docker 1.10及之后的版本是正常的。

    $ ls -l /var/lib/docker/overlay/

    total 20
    drwx------ 3 root root 4096 Jun 20 16:11 38f3ed2eac129654acef11c32670b534670c3a06e483fce313d72e3e0a15baa8
    drwx------ 3 root root 4096 Jun 20 16:11 55f1e14c361b90570df46371b20ce6d480c434981cbda5fd68c6ff61aa0a5358
    drwx------ 3 root root 4096 Jun 20 16:11 824c8a961a4f5e8fe4f4243dab57c5be798e7fd195f6d88ab06aea92ba931654
    drwx------ 3 root root 4096 Jun 20 16:11 ad0fe55125ebf599da124da175174a4b8c1878afe6907bf7c78570341f308461
    drwx------ 3 root root 4096 Jun 20 16:11 edab9b5e5bf73f2997524eebeac1de4cf9c8b904fa8ad3ec43b3504196aa3801

镜像的目录包含着每一层自己的文件内容以及对下层镜像层数据的硬链接。这保证了对磁盘空间的高效利用。

    $ ls -i /var/lib/docker/overlay/38f3ed2eac129654acef11c32670b534670c3a06e483fce313d72e3e0a15baa8/root/bin/ls

    19793696 /var/lib/docker/overlay/38f3ed2eac129654acef11c32670b534670c3a06e483fce313d72e3e0a15baa8/root/bin/ls

    $ ls -i /var/lib/docker/overlay/55f1e14c361b90570df46371b20ce6d480c434981cbda5fd68c6ff61aa0a5358/root/bin/ls

    19793696 /var/lib/docker/overlay/55f1e14c361b90570df46371b20ce6d480c434981cbda5fd68c6ff61aa0a5358/root/bin/ls

容器在宿主机的文件系统中也存储在`/var/lib/docker/overlay/`目录中。如果你查看关联到运行中容器的目录，你可以找到下面这些文件和目录。

    $ ls -l /var/lib/docker/overlay/<directory-of-running-container>

    total 16
    -rw-r--r-- 1 root root   64 Jun 20 16:39 lower-id
    drwxr-xr-x 1 root root 4096 Jun 20 16:39 merged
    drwxr-xr-x 4 root root 4096 Jun 20 16:39 upper
    drwx------ 3 root root 4096 Jun 20 16:39 work

这四个文件系统的对象组成OverlayFS的基础组件。"lower-id"文件包含了容器基于的镜像的基础镜像的顶层ID，被OverlayFS作为"lowerdir"使用。

    $ cat /var/lib/docker/overlay/ec444863a55a9f1ca2df72223d459c5d940a721b2288ff86a3f27be28b53be6c/lower-id

    55f1e14c361b90570df46371b20ce6d480c434981cbda5fd68c6ff61aa0a5358

而"upper"目录被用作容器的读写层。所有对容器的修改都会被写入到这一层中。

"merged"目录是容器的挂载点。这个是镜像("lowerdir")和容器("upperdir")的暴漏出的统一视图。所有的在容器中的修改会被直接写到这个文件夹中。

"work"目录是OverlayFS功能所需的。它用于做类似于*copy_up*操作。

你可以通过`mount`命令来验证这整个的结构。(省略号和换行是为了让输出更易读)


    $ mount | grep overlay

    overlay on /var/lib/docker/overlay/ec444863a55a.../merged
    type overlay (rw,relatime,lowerdir=/var/lib/docker/overlay/55f1e14c361b.../root,
    upperdir=/var/lib/docker/overlay/ec444863a55a.../upper,
    workdir=/var/lib/docker/overlay/ec444863a55a.../work)

下面的输出表明overlay是通过读写("rw")挂载的。


## 镜像的分层和共享 (`overlay2`)

`overlay`驱动只能工作于单个的OverlayFS lower 层，而且需要依赖于硬链接来实现多层镜像，而`overlay2`驱动原生的支持多个OverlayFS底层(最多到128)。

因此`overlay2`驱动为层相关的Docker命令(比如`docker build`和`docker commit`)提供了更好的性能，以及比`overlay`驱动更小的inodes开销。

### 示例：镜像和容器在磁盘上的结构 (`overlay2`)

当使用`docker pull ubuntu`下载了一个5层的镜像后，你可以在`/var/lib/docker/overlay2`的目录下看到6个目录。

    $ ls -l /var/lib/docker/overlay2

    total 24
    drwx------ 5 root root 4096 Jun 20 07:36 223c2864175491657d238e2664251df13b63adb8d050924fd1bfcdb278b866f7
    drwx------ 3 root root 4096 Jun 20 07:36 3a36935c9df35472229c57f4a27105a136f5e4dbef0f87905b2e506e494e348b
    drwx------ 5 root root 4096 Jun 20 07:36 4e9fa83caff3e8f4cc83693fa407a4a9fac9573deaf481506c102d484dd1e6a1
    drwx------ 5 root root 4096 Jun 20 07:36 e8876a226237217ec61c4baf238a32992291d059fdac95ed6303bdff3f59cff5
    drwx------ 5 root root 4096 Jun 20 07:36 eca1e4e1694283e001f200a667bb3cb40853cf2d1b12c29feda7422fed78afed
    drwx------ 2 root root 4096 Jun 20 07:36 l

”l“目录包含了到镜像间断的标识作为动态链接。这些间断的标识是为了避免导致挂载参数的大小限制。

    $ ls -l /var/lib/docker/overlay2/l

    total 20
    lrwxrwxrwx 1 root root 72 Jun 20 07:36 6Y5IM2XC7TSNIJZZFLJCS6I4I4 -> ../3a36935c9df35472229c57f4a27105a136f5e4dbef0f87905b2e506e494e348b/diff
    lrwxrwxrwx 1 root root 72 Jun 20 07:36 B3WWEFKBG3PLLV737KZFIASSW7 -> ../4e9fa83caff3e8f4cc83693fa407a4a9fac9573deaf481506c102d484dd1e6a1/diff
    lrwxrwxrwx 1 root root 72 Jun 20 07:36 JEYMODZYFCZFYSDABYXD5MF6YO -> ../eca1e4e1694283e001f200a667bb3cb40853cf2d1b12c29feda7422fed78afed/diff
    lrwxrwxrwx 1 root root 72 Jun 20 07:36 NFYKDW6APBCCUCTOUSYDH4DXAT -> ../223c2864175491657d238e2664251df13b63adb8d050924fd1bfcdb278b866f7/diff
    lrwxrwxrwx 1 root root 72 Jun 20 07:36 UL2MW33MSE3Q5VYIKBRN4ZAGQP -> ../e8876a226237217ec61c4baf238a32992291d059fdac95ed6303bdff3f59cff5/diff

最下层的镜像层包含了缩短的标识的"link"文件，以及包含镜像内容的"diff"文件夹。

    $ ls /var/lib/docker/overlay2/3a36935c9df35472229c57f4a27105a136f5e4dbef0f87905b2e506e494e348b/

    diff  link

    $ cat /var/lib/docker/overlay2/3a36935c9df35472229c57f4a27105a136f5e4dbef0f87905b2e506e494e348b/link

    6Y5IM2XC7TSNIJZZFLJCS6I4I4

    $ ls  /var/lib/docker/overlay2/3a36935c9df35472229c57f4a27105a136f5e4dbef0f87905b2e506e494e348b/diff

    bin  boot  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var

而第二层包含了"lower"文件表示下面的层的构成，以及"diff"目录包含层的内容。它也包含了"merged"和"work"目录。

    $ ls /var/lib/docker/overlay2/223c2864175491657d238e2664251df13b63adb8d050924fd1bfcdb278b866f7

    diff  link  lower  merged  work

    $ cat /var/lib/docker/overlay2/223c2864175491657d238e2664251df13b63adb8d050924fd1bfcdb278b866f7/lower

    l/6Y5IM2XC7TSNIJZZFLJCS6I4I4

    $ ls /var/lib/docker/overlay2/223c2864175491657d238e2664251df13b63adb8d050924fd1bfcdb278b866f7/diff/

    etc  sbin  usr  var

而运行中的容器的目录中拥有类似的文件和目录，注意到下层的列表用":"分隔，依次从高层到底层排列。

    $ ls -l /var/lib/docker/overlay/<directory-of-running-container>

    $ cat /var/lib/docker/overlay/<directory-of-running-container>/lower

    l/DJA75GUWHWG7EWICFYX54FIOVT:l/B3WWEFKBG3PLLV737KZFIASSW7:l/JEYMODZYFCZFYSDABYXD5MF6YO:l/UL2MW33MSE3Q5VYIKBRN4ZAGQP:l/NFYKDW6APBCCUCTOUSYDH4DXAT:l/6Y5IM2XC7TSNIJZZFLJCS6I4I4

查看`mount`的结果如下：

    $ mount | grep overlay

    overlay on /var/lib/docker/overlay2/9186877cdf386d0a3b016149cf30c208f326dca307529e646afce5b3f83f5304/merged
    type overlay (rw,relatime,
    lowerdir=l/DJA75GUWHWG7EWICFYX54FIOVT:l/B3WWEFKBG3PLLV737KZFIASSW7:l/JEYMODZYFCZFYSDABYXD5MF6YO:l/UL2MW33MSE3Q5VYIKBRN4ZAGQP:l/NFYKDW6APBCCUCTOUSYDH4DXAT:l/6Y5IM2XC7TSNIJZZFLJCS6I4I4,
    upperdir=9186877cdf386d0a3b016149cf30c208f326dca307529e646afce5b3f83f5304/diff,
    workdir=9186877cdf386d0a3b016149cf30c208f326dca307529e646afce5b3f83f5304/work)

## 在overlay驱动下的容器读与写

在overlay驱动下考虑三种容器打开文件读取的场景：

- **文件不存在在容器层**. 如果容器打开和读取了一个并不存在于容器层("upperdir")中的一个文件，它将从镜像("lowerdir")中读取，这个会造成很小的性能损耗。

- **文件只存在于容器层**. 如果一个容器打开和读取存在于容器("upperdir")而不在镜像("lowerdir")中的文件，就会直接从容器中读取。

- **文件存储在镜像层和容器层**. 如果一个容器打开和读取一个都存在于镜像和容器中的文件，那么容器层中的文件将被读取到。因为容器层("upperdir")中的文件覆盖了镜像层("lowerdir")中的文件。

考虑文件在容器中被修改的一些场景：

- **第一次修改文件**. 如果第一次修改已存在的文件，这个文件不存在与容器("upperdir")中。`overlay`/`overlay2`的驱动会执行*copy_up*的操作，将这个文件从镜像('lowerdir')中拷贝到容器("upperdir")中，然后这个容器会将修改写入到新拷贝到容器层中的文件中。

    然而，OverlayFS工作在文件上而不是磁盘块上。这意味着OverlayFS的copy-up操作拷贝整个文件。就算文件很大，但是只有很小的部分修改。这个会导致很可见影响写入性能。不过，有两件事情值得提一下：
    
    * copy_up操作只会在第一次修改文件时做。后续的对同一文件的修改会直接修改已经拷贝到容器中的文件的实例。
    * OverlayFS只会工作在两层，这意味着性能消耗会比在很多层的镜像中寻找文件的AUFS有更低的延迟。

- **删除文件和目录**. 当容器中的文件被删除时，一个*writeout*文件会在容器层("upperdir")中创建，而这个文件在镜像层("lowerdir")中并没有被删除，只是在容器中覆盖了这个文件的可见性。

    删除容器中的一个目录会在容器("upperdir")中创建一个*opaque directory*的操作，这个和一个whiteout文件的效果是一样的并有效的隐藏了镜像中的目录。

- **重命名目录**. 对一个目录调用`rename(2)`的系统调用只允许源目录和摩的目录都在上层，否则就会收到`EXDEV` ("cross-device link not permitted")的错误码。

所以你的应用需要设计为可以接收`EXDEV`错误码，并通过"copy和unlink"策略实现。

## 配置Docker 使用`overlay`/`overlay2`存储驱动

如果在你的Docker宿主机上配置使用`overlay`存储驱动，要保证你的Linux kernel是在3.18及以后的版本，并保证overlay的kernel 模块被加载了。而如果使用`overlay2`的驱动的话，你的内核需要是4.0及之后的版本。OverlayFS可以运行在目前主流的Linux文件系统上，不过ext4目前还是最推荐在生产环境使用的底层文件系统。

下面的流程将展示你如何配置你的Docker宿主机使用OverlayFS。这个流程事先需要你的Docker daemon是停止状态的。

> **警告:** 如果你之前已经在你的宿主机上使用了Docker并且有需要保存的镜像，请先将他们Push到
> Docker Hub或者你私有的Docker Trusty Registry。

1. 首先停掉Running的Docker daemon。

2. 验证你的Kernel版本以及overlay的kernel模块加载情况.

        $ uname -r

        3.19.0-21-generic

        $ lsmod | grep overlay

        overlay

3. 使用 `overlay`/`overlay2` storage driver存储驱动启动Docker daemon.

        $ dockerd --storage-driver=overlay &

        [1] 29403
        root@ip-10-0-0-174:/home/ubuntu# INFO[0000] Listening for HTTP on unix (/var/run/docker.sock)
        INFO[0000] Option DefaultDriver: bridge
        INFO[0000] Option DefaultNetwork: bridge
        <output truncated>
    
    或者，你可以通过修改Docker的配置文件中的`DOCKER_OPTS`添加`--storage-driver=overlay`标记让Docker daemon自动的使用`overlay`/`overlay2`的驱动，只要这个参数配设置了，你可以通过标准的启动脚本启动而不需要手动的添加`--storage-driver`参数。

4. 验证Docker daemon 使用了`overlay`/`overlay2` 存储驱动

        $ docker info

        Containers: 0
        Images: 0
        Storage Driver: overlay
         Backing Filesystem: extfs
        <output truncated>

    注意到这里输出的*Backing filesystem*是`extfs`。虽然后端的文件系统支持很多种，但是在生产环境还是推荐使用`extfs`(ext4)文件系统。

你的Docker宿主机目前已经使用了`overlay`/`overlay2`存储驱动。如果你运行`mount`命令，你可以发现Docker自动的通过需要的"lowdir","upperdir","merged","workdir"创建了`overlay`类型的挂载。

## OverlayFS在Docker上的性能

在一般的场景下，`overlay`/`overlay2`的驱动是比`aufs`和`devicemapper`快的。在特定的场景上，会比`btrfs`快。也就是说，在Docker上使用`overlay`/`overlay2`的性能是较好的。

- **页缓存**. OverlayFS支持页缓存共享，这意味着多个容器访问同样的文件可以共享同一个页缓存。这使得`overlay`/`overlay2`驱动有更高的内存效率。所以适合于Pass平台或者高密度的使用场景。

- **copy_up**. 和AUFS一样，OverlayFS每次写入到文件时会执行copy_up操作，在写操作的时候会导致一定的延时 &mdash; 特别是需要copy-up的文件很大时会更加严重。而对这个文件的后续的操作不会再需要copy-up操作。
    
    OverlayFS的copy_up操作比AUFS的操作更加快速。这因为AUFS支持多层的挂载，在多个AUFS层中寻找时会耗费更多的时间。
    
- **Inode 限制**. 使用`overlay`驱动会导致大量的inode的消耗。特别是在很多镜像和容器启停的时候会导致系统的inode快速的耗尽，`overlay2`就不会再存在这个问题。

不幸的是，你只能在创建文件系统创建的时候才能指定inode的数量。这样你就需要将`/var/lib/docker`存放在一个拥有自己的文件系统的单独的设备上，然后在创建文件系统的时候指定它的inode的数量。


以下这些通用的最佳实践同样适用于OverlayFS。

- **固态硬盘设备(SSD)**. 需要更好的存储性能的一个好主意就是更换更好的性能的固态硬盘存储 (SSD)。

- **使用数据卷s**. 数据卷提供最好的以及可预测的性能。这是因为它绕过存储驱动，不会增加由精简配置和copy-on-write带来的额外的性能消耗。由于这个原因，你应该将更重负载的数据读写放到数据卷中。


## OverlayFS 兼容性

总结下OverlayFS和其他文件系统不兼容的地方：

- **open(2)**. OverlayFS只实现了POSIX标准中的一个子集，这回导致OverlayFS有不兼容于POSIX标准。一个典型的操作是*copy-up*操作，假设你的程序通过调用`fd1=open("foo", O_RDONLY)`然后 `fd2=open("foo", O_RDWR)`分别以读和读写的方式打开同一个文件。在这种情况下，你的应用程序期望`fd1`和`fd2`指向同一个文件，然而，鉴于copy-up操作只发生在调用`open(2)`的时候，所以文件描述符会指向不同的文件。

`yum` 就会被这个影响，除非你安装了`yum-plugin-ovl`的包才能解决。如果你的发行版不支持`yum-plugin-ovl`的包（比如RHEL/CentOS 6.8或7.2），你需要在运行`yum install`之前先运行`touch /var/lib/rpm/*`来解决这个问题。

- **rename(2)**. OverlayFS 不能完全支持`rename(2)`的系统调用。你的应用程序需要检测它的失败，并通过"copy 和 unlink"的策略去绕过这个问题。