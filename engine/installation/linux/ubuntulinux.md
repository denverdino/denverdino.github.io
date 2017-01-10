---
description: Instructions for installing Docker on Ubuntu
keywords: Docker, Docker documentation, requirements, apt, installation,  ubuntu
redirect_from:
- /engine/installation/ubuntulinux/
- /installation/ubuntulinux/
title: Install Docker on Ubuntu
---

Docker 目前支持以下的Ubuntu系统:

- Ubuntu Yakkety 16.10
- Ubuntu Xenial 16.04 (LTS)
- Ubuntu Trusty 14.04 (LTS)
- Ubuntu Precise 12.04 (LTS)


这个页面知道你在Ubuntu系统上使用Docker提供的软件包来安装。这些软件包确保你可以获取最新版本的Docker.
如果你想通过`Ubuntu包管理器`来安装，请查阅你的Ubuntu文档。其中的一些文件和命令可能和你使用`Ubuntu包管理器`
有些不同。

>**说明**: Ubuntu 14.10, 15.10, and 15.04 目前已经存在于Docker `APT`库，但是官方已不再提供支持。

## 先决条件

Docker 有两条重要的安装要求:

- Docker仅能安装在64位的Linux系统上.
- 在低于3.10版本的内核上运行 Docker 会丢失一部分功能。在这些旧的版本上运行 Docker 会出现一些BUG，
 这些BUG在一定的条件里会导致数据的丢失，或者报一些严重的错误.

 检查当前系统的内核版本，打开命令行终端并且执行 `uname -r`命令来显示你的系统的内核版本:

  ```bash
  $ uname -r
  3.11.0-15-generic
  ```

### 更新apt源

设置 `APT` 使用 Docker 资源库中的软件包:

1.  以 `sudo` 或者 `root` 身份登陆系统.

2.  打开一个终端窗口.

3.  更新软件包, 确保`APT`是以 `https` 方式进行, 且安装了CA证书.

    ```bash
    $ sudo apt-get update
    $ sudo apt-get install apt-transport-https ca-certificates
    ```
4.  添加新的 `GPG` 秘钥. 该命令从秘钥服务器`hkp://ha.pool.sks-keyservers.net:80`下载
    ID为`58118E89F3A912897C070ADBF76221572C52609D` 的秘钥，并且添加到 `adv` 秘钥链.
    要想获取更多帮助, 请参阅 `man apt-key`.

    ```bash
    $ sudo apt-key adv \
                   --keyserver hkp://ha.pool.sks-keyservers.net:80 \
                   --recv-keys 58118E89F3A912897C070ADBF76221572C52609D
    ```

5.  从以下的表格中找到适合你的Ubuntu版本的源地址.这将决定APT从哪里查找Docker软件包.
    如果可能的话,运行一个(LTS)版本的Ubuntu.

    | Ubuntu version      | Repository                                                   |
    | ------------------- | ------------------------------------------------------------ |
    | Precise 12.04 (LTS) | `deb https://apt.dockerproject.org/repo ubuntu-precise main` |
    | Trusty 14.04 (LTS)  | `deb https://apt.dockerproject.org/repo ubuntu-trusty main`  |
    | Xenial 16.04 (LTS)  | `deb https://apt.dockerproject.org/repo ubuntu-xenial main`  |
    | Yakkety 16.10       | `deb https://apt.dockerproject.org/repo ubuntu-yakkety main` |


    >**说明**: Docker并没有提供所有架构的软件包. 目前每晚都会进行二进制包的构建，你可以从
    https://master.dockerproject.org下载它们. 要安装Docker到一个多架构系统上，
    在入口添加 `[arch=...]` 语句. 更多细节请参考
    [Debian Multiarch wiki](https://wiki.debian.org/Multiarch/HOWTO#Setting_up_apt_sources)

6.  运行如下命令, 使用占位符`<REPO>`的内容替换你操作系统的条目.

    ```bash
    $ echo "<REPO>" | sudo tee /etc/apt/sources.list.d/docker.list
    ```

7.  更新`APT` 包索引.

    ```bash
    $ sudo apt-get update
    ```

8.  校验 `APT` 是从正确的仓库拉取镜像.

    当你运行以下的命令, 每个条目返回的每个版本的Docker,可供您安装. 每条记录有会有URL
    `https://apt.dockerproject.org/repo/`. 已正确安装的版本被标记为`***`.下面是截断后的输出.

    ```bash
    $ apt-cache policy docker-engine

      docker-engine:
        Installed: 1.12.2-0~trusty
        Candidate: 1.12.2-0~trusty
        Version table:
       *** 1.12.2-0~trusty 0
              500 https://apt.dockerproject.org/repo/ ubuntu-trusty/main amd64 Packages
              100 /var/lib/dpkg/status
           1.12.1-0~trusty 0
              500 https://apt.dockerproject.org/repo/ ubuntu-trusty/main amd64 Packages
           1.12.0-0~trusty 0
              500 https://apt.dockerproject.org/repo/ ubuntu-trusty/main amd64 Packages
    ```
自此，当你运行 `apt-get upgrade`, `APT` 将拉取镜像从最新的仓库.

### Ubuntu版本

- Ubuntu Yakkety 16.10
- Ubuntu Xenial 16.04 (LTS)
- Ubuntu Trusty 14.04 (LTS)

对Ubuntu Trusty, Yakkety, and Xenial版本而言, 强烈建议安装`linux-image-extra-*` 内核软件包. 
`linux-image-extra-*` 软件包将允许你使用`aufs`的存储驱动.

安装 `linux-image-extra-*` :

1.  在你的 Ubuntu 宿主机打开命令终端.

2.  升级包管理器.

    ```bash
    $ sudo apt-get update
    ```

3.  安装推荐的软件包.

    ```bash
    $ sudo apt-get install linux-image-extra-$(uname -r) linux-image-extra-virtual
    ```

4.  去阅读 [安装 Docker](ubuntulinux.md#install).

#### Ubuntu Precise 12.04 (LTS)

对于 Ubuntu Precise版本, Docker 需要 3.13 的Linux内核版本. 
安装Docker需要内核在3.13及以上版本。如果你的内核版本低于3.13你需要升级你的内核。 
通过下边的表，请查阅下边的表来确认你的环境需要哪些包:

| Package                           | Description |
| --------------------------------- | ----------- |
| `linux-image-generic-lts-trusty`  | Generic Linux kernel image. This kernel has AUFS built in. This is required to run Docker. |
| `linux-headers-generic-lts-trusty`| Allows packages such as ZFS and VirtualBox guest additions which depend on them. If you didn't install the headers for your existing kernel, then you can skip these headers for the"trusty" kernel. If you're unsure, you should include this package for safety. |
| `xserver-xorg-lts-trusty`         | Optional in non-graphical environments without Unity/Xorg. **Required** when running Docker on machine with a graphical environment. |
| `ligbl1-mesa-glx-lts-trusty`      | To learn more about the reasons for these packages, read the installation instructions for backported kernels, specifically the [LTS Enablement Stack](https://wiki.ubuntu.com/Kernel/LTSEnablementStack). Refer to note 5 under each version. |


请按照以下步骤来升级你的内部版本以及相应的软件包:

1.  在Ubuntu 宿主机打开一个命令终端.

2.  升级包管理器.

    ```bash
    $ sudo apt-get update
    ```

3.  安装必须的以及可选的软件包.

    ```bash
    $ sudo apt-get install linux-image-generic-lts-trusty
    ```

    重复以上的步骤来完成其他需要的软件包的安装.
    
4.  重启系统使得新的内核生效.

    ```bash
    $ sudo reboot
    ```

5.  系统重启后, 去查看[安装 Docker](ubuntulinux.md#install).

## 安装最新版本

请确保满足所有的[先决条件](ubuntulinux.md#prerequisites), 然后按照如下步骤进行安装.

>**说明**: 对于生产系统, 强烈建议安装[安装特定版本](ubuntulinux.md#install-a-specific-version) 
因为你不愿意升级Docker。你应该仔细的指定升级生产系统的计划.

1.  以 `sudo` 权限登陆到你的Ubuntu系统.

2.  更新 `APT` 索引.

    ```bash
    $ sudo apt-get update
    ```
3.  安装 Docker.

    ```bash
    $ sudo apt-get install docker-engine
    ```

4.  启动 `docker` 守护进程.

    ```bash
    $ sudo service docker start
    ```

5.  通过运行`hello-world`镜像校验 `docker` 是否正确安装.

    ```bash
    $ sudo docker run hello-world
    ```

    该命令下载一个测试镜像且运行在容器里。当容器运行起来后，会打印一条信息，然后，退出。

## 安装特定版本

安装特定版本的 `docker-engine`:

1.  使用 `apt-cache madison`列举所有可用的版本:
    ```bash
    $ apt-cache madison docker-engine

    docker-engine | 1.12.3-0~xenial | https://apt.dockerproject.org/repo ubuntu-xenial/main amd64 Packages
    docker-engine | 1.12.2-0~xenial | https://apt.dockerproject.org/repo ubuntu-xenial/main amd64 Packages
    docker-engine | 1.12.1-0~xenial | https://apt.dockerproject.org/repo ubuntu-xenial/main amd64 Packages
    docker-engine | 1.12.0-0~xenial | https://apt.dockerproject.org/repo ubuntu-xenial/main amd64 Packages
    docker-engine | 1.11.2-0~xenial | https://apt.dockerproject.org/repo ubuntu-xenial/main amd64 Packages
    docker-engine | 1.11.1-0~xenial | https://apt.dockerproject.org/repo ubuntu-xenial/main amd64 Packages
    docker-engine | 1.11.0-0~xenial | https://apt.dockerproject.org/repo ubuntu-xenial/main amd64 Packages
    ```

2.  第二个字段是版本号. 如果要安装 `1.12.0-0~xenial`版本,
    追加版本到 `apt-get install` 命令后边的软件包名称后边，
    并且使用等号(`=`)分隔软件包名称和版本号.
    ```bash
    $ sudo apt-get install docker-engine=1.12.0-0~xenial
    ```

    
    如果你已经安装了一个新版本,系统将提示您降低Docker版本。否则,将安装特定版本。

3.  参考步骤4和5[安装最新版本](ubuntulinux.md#install-the-latest-version).

## 安装预发布版本


如果你想在非生产环境的Ubuntu系统上测试Docker,遵循这些步骤。
安装一个稳定的发布版本的Docker之后,您将需要回到之前的配置。


1.  编辑 `/etc/apt/sources.list.d/docker.list`.

    ```bash
    $ sudo nano /etc/apt/sources.list.d/docker.list
    ```

    把最上方那行结束部分的数据从 `main` 改成 `testing` . 保存并关闭文件.

2.  更新软件包.

    ```bash
    $ sudo apt-get update
    ```

3.  查看可用的测试版本.

    ```bash
    $ sudo apt-cache madison docker-engine
    ```

4.  安装特定版本的过程是相同的[安装特定版本](ubuntulinux.md#install-a-specific-version).

## 可选配置

这个部分包含的可选程序来配置Ubuntu，使得Docker工作的更好.

* [使用非root账户](ubuntulinux.md#manage-docker-as-a-non-root-user)
* [调整内存和交换空间](ubuntulinux.md#adjust-memory-and-swap-accounting)
* [允许UFW端口转发](ubuntulinux.md#enable-ufw-forwarding)
* [Docker配置DNS服务](ubuntulinux.md#configure-a-dns-server-for-use-by-docker)
* [配置Docker开机启动](ubuntulinux.md#configure-docker-to-start-on-boot)

### 使用非root账号

`docker`守护进程是以`root`用户身份运行，并且 `docker`守护进程使用Unix socket监听来
替代TCP端口监听. 默认情况下，Unix socket 属于 `root`用户, 当然其他用户也可以通过 `sudo`方式来访问.

如果你 (或者你安装Docker的时候) 创建一个叫 `docker`的用户组，并且为用户组添加用户.
这时候，当`docker` 守护进程启动的时候, `docker` 用户组对Unxi Socket有了读/写权限.
你必须使用root用户来运行`docker`守护进程, 但是你可以使用`docker`群组用户来使用
`docker` 客户端，你在使用 `docker` 命令的时候前边就不需要添加 `sudo`了。 从 Docker 0.9.0 
版本开始，你可以使用`-G` 来指定用户组.

>**警告**: `docker` 用户组 (或者用 `-G` 指定的用户组) 等同于
`root`用户的权限; 有关安全影响的细节，请查看 [*Docker进程表面攻击细节*](../../security/security.md#docker-daemon-attack-surface).

To create the `docker` group and add your user:

1.  以 `sudo` 身份登陆Ubuntu系统.

2.  创建 `docker` 用户组.
    ```bash
    $ sudo groupadd docker
    ```

3.  添加用户到 `docker` 用户组.

    ```bash
    $ sudo usermod -aG docker $USER
    ```

4.  重新登陆使得你的组关系生效.

5.  校验你是否可以直接运行 `docker` 命令，而无需在添加 `sudo`.

    ```bash
    $ docker run hello-world
    ```

	  如果以上失败，你将看到如下错误:

    ```none
		Cannot connect to the Docker daemon. Is 'docker daemon' running on this host?
    ```

	  检查在当前会话中是否设置`DOCKER_HOST`的环境变量.

    ```bash
    $ env | grep DOCKER_HOST
    ```

      如果环境变量已设置，以上命令会返回设置的结果。否则重置它.

    ```bash
    $ unset DOCKER_HOST
    ```

    你可能需要编辑环境变量在文件`~/.bashrc` 或`~/.profile`中来防止 `DOCKER_HOST`
    环境变量被错误的设置.

### 调整内存和交换空间

当我们使用 Docker 运行一个镜像的时候，我们可能会看到如下的信息提示:

```none
WARNING: Your kernel does not support cgroup swap limit. WARNING: Your
kernel does not support swap limit capabilities. Limitation discarded.
```

如果你不太关心这些特性，可以忽略以上警告。当然，你可以启用这些新的特性在参考一下步骤.
即使在Docker没有运行的情况下，内存和交换空间产生的开销约占总数的1%内存和整体性能下降10%

1.  以 `sudo` 身份登录到你的Ubuntu系统.

2.  编辑 `/etc/default/grub` 文件.

3.  添加或者编辑`GRUB_CMDLINE_LINUX` 行，添加以下两对key-value的数据:

    ```none
    GRUB_CMDLINE_LINUX="cgroup_enable=memory swapaccount=1"
    ```

    保存并关闭文件.

4.  更新 GRUB.

    ```bash
    $ sudo update-grub
    ```
     
     如果你的GRUB配置文件有格式错误，将会引发错误。在这种情况下，请检查步骤3和步骤4.

6.  重启系统，内存和交换空间被启用，且没有警告发生.


### 允许UFW端口 转发

当你在运行 docker 的宿主主机上使用[UFW (简单防火墙)](https://help.ubuntu.com/community/UFW)。
你需要做一些额外的配置。Docker 使用桥接的方式来管理网络。默认情况下，UFW 过滤所有的端口转发策略。
因此，你必须适当的设置UFW的端口转发策略。

默认情况下UFW是过滤掉所有的入站规则。如果你想从另外的主机访问Docker 远程API
和启动远程访问，你需要配置UFW允许Docker默认端口`2376`(如果启用TLS加密传输) 或者 `2375`的所有连接。
默认情况下，Docker并不启用TLS，如果你不使用TLS，你应该极力阻止从远程主机访问Docker 远程API，以
阻止远程越权攻击.

设置 UFW 允许Docker 端口的入站规则：

1.  使用具有`sudo`权限的用户来登录你的Ubuntu。

2.  校验UFW是否启用.

    ```bash
    $ sudo ufw status
    ```

    如果`ufw`没有被启动，剩下的步骤将是无用的。

3.  编辑 UFW 配置文件, 通常位于 `/etc/default/ufw` 或者
`/etc/sysconfig/ufw`. 设置 `DEFAULT_FORWARD_POLICY` 策略为 `ACCEPT`.

    ```none
    DEFAULT_FORWARD_POLICY="ACCEPT"
    ```

    保存并关闭文件.

4.  如果你需要启用从外部主机访问Docker远程API，并且了解安全影响
     (在之前的部分),然后配置UFW策略允许来源请求访问Docker端口，
     如果未启用TLS为`2375`端口，否则为`2376`端口

    ```bash
    $ sudo ufw allow 2376/tcp
    ```

5.  重新加载 UFW.
    ```bash
    $ sudo ufw reload
    ```

### Docker配置DNS服务

使用`networkmanager`的Ubuntu系统使用`dnsmasq`上运行回环地址如`127.0.0.1` 或 `127.0.1.1`
并且添加该条目到`/etc/resolv.conf`.`dnsmasq`服务提供本地的DNS缓存加速DNS查询,还提供DHCP服务.
这个配置在有自己网络命名空间的Docker容器里将不能工作.这是因为Docker容器把像`127.0.0.1`这样的回环
地址解析到自身,也不太可能运行在自身的回环地址上运行一个DNS服务器.


如果Docker检测到没有一个全功能的DNS服务器在`/etc/resolv.conf`中被引用,在Docker使用由谷歌提供的
`8.8.8.8` 和 `8.8.4.4` 公网DNS服务器做DNS解析的时候会发生以下警告.

```none
WARNING: Local (127.0.0.1) DNS resolver found in resolv.conf and containers
can't use it. Using default external servers : [8.8.8.8 8.8.4.4]
```

如果你没有使用 `dnsmasq`或者NetworkManaager或从未见过以上的警告信息，你可以跳过该章节的
剩余部分. 使用以下命令查看是否使用`dnsmasq`:

```bash
$ ps aux |grep dnsmasq
```

如果发生这些警告并且不能使用公网的域名服务器,当你需要运行一个在内部网络中解析域名的DNS服务器时，
你有以下两种选择:

- 你可以为Docker指定DNS服务器.
- 你可以在NetworkManager中禁用 `dnsmasq`. 如果你这样做,NetworkManager将添加你真是的DNS命名服务器
 到`/etc/resolv.conf`文件，但是你将失去`dnsmasq`可能带来的好处.

**你只需要使用这些方法之一**

#### 为Docker指定DNS服务器

以下的说明工作在你的Ubuntu系统安装了upstart` 或者`systemd`.

默认的配置文件在`/etc/docker/daemon.json`位置.你可以使用`--config-file`标签来改变配置文件
的位置.下面的文档假设配置文件存放在 `/etc/docker/daemon.json`.

1.  使用具有`sudo`权限的用户来登录你的Ubuntu。

2.  创建或者编辑Docker的配置文件，默认在`/etc/docker/daemon.json`， 它控制Docker的配置。

    ```bash
    sudo nano /etc/docker/daemon.json
    ```

2.  新增 `dns`对应的一个或者多个IP的键值对，如果文件已存在相同的内容，
    你仅需要添加或者编辑`dns` 行.
    ```json
    {
    	"dns": ["8.8.8.8", "8.8.4.4"]
    }
    ```

    如果你的内部DNS服务器不能解析公网IP地址，至少需要一个DNS服务器可以解析公网域名,这样
    你可以连接到Docker Hub且你的容器能解析公网的域名.

    保存并关闭文件.

3.  重启Docker.

    ```bash
    $ sudo service docker restart
    ```

4.  通过拉取镜像可以测试Docker是否可以解析外部的IP地址:

    ```bash
    $ docker pull hello-world
    ```

5.  如果有必要的话，可以通过ping命令在检查Docker容器解析内部主机名.

    ```bash
    $ docker run --rm -it alpine ping -c4 my_internal_host

    PING google.com (192.168.1.2): 56 data bytes
    64 bytes from 192.168.1.2: seq=0 ttl=41 time=7.597 ms
    64 bytes from 192.168.1.2: seq=1 ttl=41 time=7.635 ms
    64 bytes from 192.168.1.2: seq=2 ttl=41 time=7.660 ms
    64 bytes from 192.168.1.2: seq=3 ttl=41 time=7.677 ms
    ```

#### 在NetworkManager禁用 `dnsmasq` 

如果你不愿意使用一个特定的IP地址来修改Docker进程的配置文件,参考以下说明在NetworkManager中禁用`dnsmasq`.

1.  编辑文件 `/etc/NetworkManager/NetworkManager.conf` .

2.   在 `dns=dnsmasq` 前加 `#` 注释掉该行.

    ```none
    # dns=dnsmasq
    ```

    保存并关闭文件.

4.  重启NetworkManager 和 Docker. 作为替代方式，你可以直接重启系统.

    ```bash
    $ sudo restart network-manager
    $ sudo restart docker
    ```

### 配置Docker开机启动

自`15.04`起，Ubuntu使用`systemd`作为自启动以及服务的管理器，而且`14.10`以及早期版本使用`upstart`.

#### `systemd`

```bash
$ sudo systemctl enable docker
```

#### `upstart`

`14.10`以及早期的版本，Docker通过`upstart` 配置开机自动启动.

## 升级 Docker

可以通过使用`ap-get`来安装最新版本的Docker.下边的示例演示了获取所有系统可用的版本，
然后升级Docker如果新版本存在的话.

```bash
$ sudo apt-get update
$ sudo apt-get upgrade docker-engine
```

## 卸载

卸载Docker软件包:

```bash
$ sudo apt-get purge docker-engine
```

卸载Docker软件包以及不在被使用的依赖的软件包:

```bash
$ sudo apt-get autoremove --purge docker-engine
```

镜像，容器，数据卷以及自定义的配置文件不能被自动移除。可以通过运行以下的命令来删除镜像，容器和数据卷:

```bash
$ rm -rf /var/lib/docker
```

你必须手动删除编辑过的配置文件。
