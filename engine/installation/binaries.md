---
description: Instructions for installing Docker as a binary. Mostly meant for hackers who want to try out Docker on a variety of environments.
keywords: binaries, installation, docker, documentation, linux
title: Install Docker from binaries
---

**这份说明主要供那些想在不同环境中体验Docker的hacker们参考**

在安装之前，请检查你的 Linux 发行版本是否有打包好的 Docker 安装包。我们已经发布了许多发行版包，这样会节省您很多时间。

## 检查运行时的依赖关系

如果想要 Docker 正常运行，需要安装以下软件:

 - iptables version 1.4 or later
 - Git version 1.7 or later
 - procps (or similar provider of a "ps" executable)
 - XZ Utils 4.9 or later
 - a [properly mounted](
   https://github.com/tianon/cgroupfs-mount/blob/master/cgroupfs-mount)
   cgroupfs hierarchy (having a single, all-encompassing "cgroup" mount
   point [is](https://github.com/docker/docker/issues/2683)
   [not](https://github.com/docker/docker/issues/3485)
   [sufficient](https://github.com/docker/docker/issues/4568))

## 检查内核的依赖关系

Docker 在守护进程模式中需要特定的内核支持。具体要求请参考[*安装说明*](index.md#on-linux).

Docker要求最低Linux版本为3.10。
内核低于3.10的系统缺少一些特性来满足Docker容器运行要求，这些低版本系统在某些条件下有缺陷，如造成数据丢失或频繁报错。

推荐使用版本号为（3.x.y）的 3.10 Linux 内核版本（或者更新的维护版本），保持内核的版本同步更新，这样能够保证内核的BUG已经被修复。

> **警告**:
> Linux 版本发行商可能不支持安装自定义内核及软件包。请务必在安装自定义内核之前，先咨询发行商是否支持 Docker。

> **警告**:
> 一些发行版更新内核仍可能不满足需要，因为这些版本提供的软件包太老或者软件包与新内核不兼容。

值得注意的是 Docker 可以以客户端模式存在，它几乎可以运行在任何的Linux内核（甚至 macOS）上。

## 开启 AppArmor 和 SELinux

如果你的 Linux 发行版上支持 AppArmor 或 Selinux 请启用。这有助于提高安全性并阻止某些漏洞。请在发行版提供的文档中查看如何开启推荐的安全设置的详细操作。

某些 Linux 发行版默认开启了 AppArmor 或者 Selinux，但是它们的内核不符合安装 Docker 的最低要求（3.10或更高版本）。
这种情况更新内核到3.10或者更高版本并不能保证Docker可在上面启动并运行容器。
因为系统提供的AppArmor/SELinux用户空间实用工具版本同内核版本不兼容，这可能会阻止 Docker 的运行，容器的启动或者造成容器的意外退出。

> **警告**:
> 如果机器上开启了安全机制，不应该为了使用Docker和运行容器而禁用它们。
> 这样做会降低系统安全性，失去发行版供应商的服务支持，并可能破坏严格监管环境下的管理和安全策略。

## 获取Docker Engine二进制文件

你可以下载最新版本或者特定版本的二进制版本。参见`docker/docker` [Releases page](https://github.com/docker/docker/releases).

Docker每个版本的发布记录底部都会有一组下载链接。链接中包含了该版本的源代码地址、已支持平台的二进制安装文件地址和不在支持的Linux版本中的静态二进制文件地址。
可以再下载章节中找到链接列表，找到合适的二进制文件来下载。

### Windows 和 macOS 二进制文件的限制

对于Windows系统，`i386` 下载仅是一个32位客户端安装包。`x86_64` 下载文件中包含客户端和服务端安装文件，可以在64位Windows
Server 2016 和 Windows 10 中使用.

macOS二进制文件仅包含客户端，你不能使用它来启用`dockerd`进程。如果要运行守护进程，请安装 [Docker for Mac](/docker-for-mac/index.md)

### 静态二进制文件的URL规则

已经发布的二进制文件的URL是固定的并遵照一定的规则。目前还不支持目录结构查看。如果你不想在发布记录中查找下载链接，可以根据下面说明的规则拼出下载URL：

| Description            | URL pattern                                                       |
|------------------------|-------------------------------------------------------------------|
| Latest Linux 64-bit    | `https://get.docker.com/builds/Linux/x86_64/docker-latest.tgz`    |
| Latest Linux 32-bit    | `https://get.docker.com/builds/Linux/i386/docker-latest.tgz`      |
| Specific version Linux 64-bit| `https://get.docker.com/builds/Linux/x86_64/docker-<version>.tgz` |
| Specific version Linux 32-bit| `https://get.docker.com/builds/Linux/i386/docker-<version>.tgz`   |
| Latest Windows 64-bit | `https://get.docker.com/builds/Windows/x86_64/docker-latest.zip`     |
| Latest Windows 32-bit | `https://get.docker.com/builds/Windows/i386/docker-latest.zip`      |
| Specific version Windows 64-bit | `https://get.docker.com/builds/Windows/x86_64/docker-<version>.zip` |
| Specific version Windows 32-bit | `https://get.docker.com/builds/Windows/i386/docker-<version>.zip` |
| Latest MacOS 64-bit   | `https://get.docker.com/builds/Darwin/x86_64/docker-latest.tgz` |
| Specific version MacOS 64-bit | `https://get.docker.com/builds/Darwin/x86_64/docker-<version>.tgz` |

例如，Docker 1.11.0 64-bit的Linux二进制安装文件地址为
`https://get.docker.com/builds/Linux/x86_64/docker-1.11.0.tgz`.


> **注意** These instructions are for Docker Engine 1.11 and up. Engine 1.10 and
> 上述规则只适用于Docker Engine 1.11及以上版本。对于Engine 1.10及以下版本的二进制文件在一个包内，规则是不同的。
> 安装1.10及以下版本请参考这些说明[1.10 documentation](https://docs.docker.com/v1.10/engine/installation/binaries/){:target="_blank"}.

#### 验证下载文件

为了验证下载文件的完整性，你可以在URL后面添加`.md5` 或 `.sha256`来获取该文件的MD5值 或 SHA256值。
例如，验证上面链接中的`docker-1.11.0.tgz`文件，可以使用URL
`https://get.docker.com/builds/Linux/x86_64/docker-1.11.0.tgz.md5` 或
`https://get.docker.com/builds/Linux/x86_64/docker-1.11.0.tgz.sha256`.

## 安装Linux二进制文件

下载并解压后，当前目录下有一个名为`docker`的文件夹，二进制文件都在这个文件夹内

```bash
$ tar -xvzf docker-latest.tgz

docker/
docker/docker
docker/docker-containerd
docker/docker-containerd-ctr
docker/docker-containerd-shim
docker/docker-proxy
docker/docker-runc
docker/dockerd
```

Engine要求这些二进制文件安装在主机的`$PATH`下。例如，安装到`/usr/bin`：

```bash
$ mv docker/* /usr/bin/
```

> **注意**: 如果你已经安装过Engine，请在安装前停掉Engine(`killall docker`)，并将文件安装在相同的位置。
> 你可以使用`dirname $(which docker)`查看docker当前的安装位置

### Linux中运行Engine daemon

你可以手动启动Engine，让其运行在守护进程模式：

```bash
$ sudo dockerd &
```

Git仓库中提供了一些初始化脚本示例，提供了使用诸如upstart和systemd的进程管理工具管理daemon的方法。
脚本地址<a href="https://github.com/docker/docker/tree/master/contrib/init">
contrib directory</a>.

想要了解更多运行Engine守护进程模式的信息，参见Engine命令行参考中的[daemon command](../reference/commandline/dockerd.md)

## macOS中二进制文件安装

双击下载的 `.tgz` 文件或通过命令行 `tar -xvzf docker-1.11.0.tgz` 来解压文件。你可以再任何位置运行客户端。

## Windows中安装二进制文件

双击下载的 `.zip` 文件来解压。你可以再任何位置运行客户端。

## 非root权限执行客户端命令

Linux中`dockerd` 守护进程由root用户运行并绑定一个Unix socket（来替代TCP端口）。
默认的，这个Unix socket由 `root` 来管理，也就是说默认你要使用 `sudo` 来运行 `docker` 命令。

如果你（或Docker安装者）创建一个叫 `docker` 的 Unix 群组，并且把用户加入到这个组中。
当进程启动的时候，`docker` 群组将有 `dockerd` 进程Unix socket的读/写权限。
`dockerd` 进程必须使用root用户运行，但是非root用户也可以使用 `docker` 客户端命令，如 `docker run`。

> **警告**:
> *docker* 用户组 (或用 `-G` 指定的用户组) 同root等效。
> 参见[*Docker Daemon Attack Surface*](../security/security.md#docker-daemon-attack-surface).

## 更新Docker Engine

Linux中升级你手动安装的 Docker Engine ,需要先停掉docker进程：

```bash
$ killall dockerd
```

Docker进程停止后，移除旧的安装文件并按着[步骤](binaries.md#get-the-linux-binaries)重新安装

## 接下来

继续阅读[用户指南](../userguide/index.md).