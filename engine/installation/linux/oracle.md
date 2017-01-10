---
description: Installation instructions for Docker on Oracle Linux.
keywords: Docker, Docker documentation, requirements, linux, rhel, centos, oracle,  ol
redirect_from:
- /engine/installation/oracle/
title: Install Docker on Oracle Linux
---

Docker 支持 Oracle Linux 6 and 7. 你不需要为在Oracle Linux上安装Docker去订阅Oracle Linux支持

## 前提

因为目前Docker的限制, 它只能在x86_64架构下运行。
Docker 需要使用 Unbreakable Enterprise Kernel Release 4 (4.1.12) 或更高版本内核的Oracle Linux. 
Oracle Linux 6 和 7 系统中该内核支持 Docker btrfs 存储引擎.

## 安装


> **注意**: 下面安装过程中使用的二进制文件由Docker构建. 这些文件并没有被Oracle Linux支持完全覆盖到。
> 为保证Oracle Linux支持, 请遵照[Oracle Linux 文档](https://docs.oracle.com/en/operating-systems/?tab=2)提供的安装说明
>
> Oracle Linux 6 和 7 的安装说明在[Chapter 2 of the Docker User&apos;s Guide](https://docs.oracle.com/cd/E52668_01/E75728/html/docker_install_upgrade.html)


1. 使用 `sudo` 或 `root` 权限用户登录主机

2. 确保已有 yum 包是最新的

        $ sudo yum update

3. 添加 yum repo

    对于Oracle Linux 6:

        $ sudo tee /etc/yum.repos.d/docker.repo <<-EOF
        [dockerrepo]
        name=Docker Repository
        baseurl=https://yum.dockerproject.org/repo/main/oraclelinux/6
        enabled=1
        gpgcheck=1
        gpgkey=https://yum.dockerproject.org/gpg
        EOF

    对于Oracle Linux 7:

        $ cat >/etc/yum.repos.d/docker.repo <<-EOF
        [dockerrepo]
        name=Docker Repository
        baseurl=https://yum.dockerproject.org/repo/main/oraclelinux/7
        enabled=1
        gpgcheck=1
        gpgkey=https://yum.dockerproject.org/gpg
        EOF

4. 安装Docker 包

        $ sudo yum install docker-engine

5. 启动 Docker 进程.

     Oracle Linux 6中:

        $ sudo service docker start

     Oracle Linux 7中:

        $ sudo systemctl start docker.service

6. 验证 `docker` 安装正确，在容器中运行下面的测试镜像

        $ sudo docker run hello-world

## 可选配置

这部分包括一些可选的Oracle Linux配置，这些会使Docker在系统中运行更顺畅.

* [创建 docker 用户组](oracle.md#create-a-docker-group)
* [配置Docker开机启动](oracle.md#configure-docker-to-start-on-boot)
* [使用btrfs存储引擎](oracle.md#use-the-btrfs-storage-engine)

### 创建Docker用户组

`docker` 进程绑定一个Unix socket来替代TCP端口. 
默认情况下Unix socket归`root`用户所有，其他用户需要使用`sudo`来获得权限
因此, `docker` 进程总是已`root` 用户身份运行.


为了避免使用`docker`命令时总是输入`sudo`，可以创建一个叫 `docker`的Unix用户组，将用户添加到组内。
当`docker` 进程启动时, `docker`用户组拥有Unix socket的读写权限

>**警告**: `docker` 用户组相当于 `root` 用户; 想要了解这对于系统安全影响的更多内容，参见
>[*Docker Daemon Attack Surface*](../../security/security.md#docker-daemon-attack-surface).

创建 `docker` 用户组并添加用户:

1. 用`sudo`权限登录Oracle Linux主机.

2. 创建 `docker` 用户组.

        $ sudo groupadd docker

3. 添加用户到 `docker` 组.

        $ sudo usermod -aG docker username

4. 退出并重新登录.

        这样保证用户获得正确权限

5. 验证是否生效，不使用`sudo`运行 `docker`.

        $ docker run hello-world

	如果出现失败提示，类似如下信息:

		Cannot connect to the Docker daemon. Is 'docker daemon' running on this host?

	检查 `DOCKER_HOST` 环境变量是否在shell中进行过设置，如果设置过，清除它
	

### 配置Docker开机启动

你可以配置  Docker 进程开机自动启动.

Oracle Linux 6中:

```
$ sudo chkconfig docker on
```

Oracle Linux 7中:

```
$ sudo systemctl enable docker.service
```

如果你想要添加一个 HTTP 代理，需要为 Docker 运行文件设置不同的目录或分区。
如果需要定制一些其它的功能，请阅读我们的systemd文章，了解如何[定制Docker进程](../../admin/systemd.md)

### 使用btrfs存储引擎

Oracle Linux 6 和 7 支持Docker使用btrfs 存储引擎.
在开启btrfs前, 请确认 `/var/lib/docker` 存储在btrfs格式的文件系统中. 
如何创建和挂载btrfs文件请参考 [Oracle Linux Administrator's Solution Guide](http://docs.oracle.com/cd/E37670_01/E37355/html/index.html)中[Chapter 5](http://docs.oracle.com/cd/E37670_01/E37355/html/ol_btrfs.html)


在Oracle Linux中开启btrfs:

1. 确保 `/var/lib/docker` 存在btrfs文件系统中.

2. 编辑 `/etc/sysconfig/docker` ， 在`OTHER_ARGS`项中添加 `-s btrfs`.

3. 重启Docker 进程:

## 卸载

卸载 Docker 包:

    $ sudo yum -y remove docker-engine

上述命令不会删除主机上的镜像，容器，数据卷及用户配置文件。如果想删除镜像，容器，数据卷可以使用下面命令：

    $ rm -rf /var/lib/docker

用户配置文件需要手动查找删除。

## 已知问题

### Docker 停止时卸载 btrfs 文件
如果你使用btrfs存储引擎运行Docker，当停止Docker服务时，btrfs文件会被卸载
在重启Docker服务前，要保证文件已挂载

在Oracle Linux 7中, 可以使用 `systemd.mount` ，修改Docker `systemd.service`，让其依赖于btrfs mount defined in systemd.

### Oracle Linux 7中SElinux设置
Oracle Linux 7 中 `/etc/sysconfig/selinux`目录下 SElinux 必须设置 `Permissive` 或 `Disabled` 才能使用btrfs存储引擎.

## 更多问题?

如果你购买了Oracle Linux Basic或Premier服务支持，在安装Docker时遇到问题可以通过[My Oracle Support](https://support.oracle.com)提问并获取帮助

如果你没有购买Oracle Linux 服务支持, 可以在[Oracle Linux Forum](https://community.oracle.com/community/server_%26_storage_systems/linux/oracle_linux)中得到社区支持.