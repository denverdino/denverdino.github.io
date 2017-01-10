---
description: Example of a manual install of Docker Engine on a cloud provider, using Amazon Web Services (AWS) EC2. Shows how to create an EC2 instance, and install Docker Engine on it.
keywords: cloud, docker, machine, documentation, installation, AWS, EC2
title: 'Example: Manual installation on a cloud provider'
---

你可有在云服务商的主机上直接安装Docker Engine. 下面这个示例介绍如何创建 <a href="https://aws.amazon.com/" target="_blank"> 亚马逊云服务 (AWS)</a> EC2 实例, 并在上面安装Docker Engine.

其他云服务商安装步骤与其类似。

### 步骤 1. 注册AWS

1. 如果你不是AWS用户, 登录 <a href="https://aws.amazon.com/" target="_blank"> AWS</a> 创建一个账号，并使用root权限登录EC2主机。如果你已经注册了AWS账号, 可以使用它作为root账号.

2. 创建一个 IAM (Identity and Access Management) 管理员用户, 管理员用户组, 以及同地域关联的键值对.

    在AWS菜单中, 选择 **Services** > **IAM** .

    查阅 AWS 文档中 <a href="http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/get-set-up-for-amazon-ec2.html" target="_blank">安装 Amazon EC2</a>部分. 按指导 "Create an IAM User" 及 "Create a Key Pair".

    如果你刚接触 AWS 和 EC2, 不要创建虚拟专有网络 (VPC) 或 设置子网. 新的 EC2-VPC 平台 (2013-12-04后创建的) 会在每个可用区提供 默认 VPC 和 子网. 当启动实例时会使用默认的VPC.


### 步骤 2. 配置并启动 EC2 实例

按着下面的步骤启动实例，使用指定的操作系统创建虚拟机：

  1. 使用你的IAM证书登录AWS.

      在AWS主页, 点击 **EC2** 前往控制台, 然后点击 **Launch Instance**.

      ![EC2 dashboard](../images/ec2_launch_instance.png)

      AWS EC2 虚拟服务器称为 *实例*. 当设置好账号, IAM和键值对后, 你就准备好启动一个实例. 这时你还需要为虚拟机选择一个操作系统.

  2. 选择 一个包含OS和应用的Amazon 机器镜像 (AMI). 这个例子中, 我们使用Ubuntu.

      ![Launch Ubuntu](../images/ec2-ubuntu.png)

  3. 选择实例类型.

      ![Choose a general purpose instance type](../images/ec2_instance_type.png)

  4. 配置实例.

    你可以选择默认的网络和子网，即地域和可用区的默认配置.

      ![Configure the instance](../images/ec2_instance_details.png)

  5. 点击 **Review and Launch**.

  6. 选择一个实例使用的键值对.

    当实例启动时, 你需要选择一个键值对. 保存好 `.pem` 文件，接下来的步骤中还会使用.

这时实例已经启动起来. 可以通过菜单操作查看EC2 实例: **EC2 (Virtual Servers in Cloud)** > **EC2 Dashboard** > **Resources** > **Running instances**.

想查看私钥文件, 实例IP地址, 如何通过SSH登陆到实例, 在AWS实例控制台顶部点击 **Connect** 按钮.


### 步骤 3. 使用终端登录，配置apt获取软件包

1. 使用终端通过命令行登录EC2实例.

    进入SSH key文件所在目录执行下面命令（或者在命令中包含目录信息）

        $ ssh -i "YourKey" ubuntu@xx.xxx.xxx.xxx

    在我们的例子中:

        $ cd ~/Desktop/keys/amazon_ec2
        $ ssh -i "my-key-pair.pem" ubuntu@xx.xxx.xxx.xxx

    We'll follow the instructions for installing Docker on Ubuntu [the instructions for installing Docker on Ubuntu](/engine/installation/linux/ubuntulinux.md). The next few steps reflect those instructions.
    我们将按照 [Ubuntu中安装Docker说明](/engine/installation/linux/ubuntulinux.md) 中的介绍在Ubuntu中安装Docker，下述是详细的安装步骤。

2. 检查内核版本，确保是3.10 或 更高的版本

        ubuntu@ip-xxx-xx-x-xxx:~$ uname -r
        3.13.0-48-generic

3. 添加新的 `gpg` key.

        ubuntu@ip-xxx-xx-x-xxx:~$ sudo apt-key adv --keyserver hkp://p80.pool.sks-keyservers.net:80 --recv-keys 58118E89F3A912897C070ADBF76221572C52609D
        Executing: gpg --ignore-time-conflict --no-options --no-default-keyring --homedir /tmp/tmp.jNZLKNnKte --no-auto-check-trustdb --trust-model always --keyring /etc/apt/trusted.gpg --primary-keyring /etc/apt/trusted.gpg --keyserver hkp://p80.pool.sks-keyservers.net:80 --recv-keys 58118E89F3A912897C070ADBF76221572C52609D
        gpg: requesting key 2C52609D from hkp server p80.pool.sks-keyservers.net
        gpg: key 2C52609D: public key "Docker Release Tool (releasedocker) <docker@docker.com>" imported
        gpg: Total number processed: 1
        gpg:               imported: 1  (RSA: 1)

4. 创建 `docker.list` 文件, 添加 Ubuntu Trusty 14.04 (LTS) 为docker源

        ubuntu@ip-xxx-xx-x-xxx:~$ sudo vi /etc/apt/sources.list.d/docker.list

   如果该文件已经存在，编辑前清除里面所有内容

5. 更新 `apt` 软件列表.

        ubuntu@ip-xxx-xx-x-xxx:~$ sudo apt-get update

6. 如果有旧的包要清理掉

    因为我们是一个新的VM，所以不需要清理。为了保证没有旧的包我们仍然可以执行下面命令

        ubuntu@ip-xxx-xx-x-xxx:~$ sudo apt-get purge lxc-docker
        Reading package lists... Done
        Building dependency tree
        Reading state information... Done
        Package 'lxc-docker' is not installed, so not removed
        0 upgraded, 0 newly installed, 0 to remove and 139 not upgraded.

7.  验证 `apt` 从新的仓库拉取.

        ubuntu@ip-172-31-0-151:~$ sudo apt-cache policy docker-engine
        docker-engine:
        Installed: (none)
        Candidate: 1.9.1-0~trusty
        Version table:
        1.9.1-0~trusty 0
        500 https://apt.dockerproject.org/repo/ ubuntu-trusty/main amd64 Packages
        1.9.0-0~trusty 0
        500 https://apt.dockerproject.org/repo/ ubuntu-trusty/main amd64 Packages
            . . .

    之后再运行 `apt-get upgrade`，`apt` 都从新的仓库拉取

### 步骤 4. 安装推荐的系统必装软件

对于 Ubuntu Trusty (及其他的一些版本), 推荐安装 `linux-image-extra` 软件包, 这样你可以使用 `aufs` 存储驱动, 执行下面命令安装.

        ubuntu@ip-xxx-xx-x-xxx:~$ sudo apt-get update
        ubuntu@ip-172-31-0-151:~$ sudo apt-get install linux-image-extra-$(uname -r)

### 步骤 5. 安装Docker Engine

1. 更新软件列表.

        ubuntu@ip-xxx-xx-x-xxx:~$ sudo apt-get update

2. 安装 Docker Engine.

        ubuntu@ip-xxx-xx-x-xxx:~$ sudo apt-get install docker-engine
        Reading package lists... Done
        Building dependency tree
        Reading state information... Done
        The following extra packages will be installed:
        aufs-tools cgroup-lite git git-man liberror-perl
        Suggested packages:
        git-daemon-run git-daemon-sysvinit git-doc git-el git-email git-gui gitk
        gitweb git-arch git-bzr git-cvs git-mediawiki git-svn
        The following NEW packages will be installed:
        aufs-tools cgroup-lite docker-engine git git-man liberror-perl
        0 upgraded, 6 newly installed, 0 to remove and 139 not upgraded.
        Need to get 11.0 MB of archives.
        After this operation, 60.3 MB of additional disk space will be used.
        Do you want to continue? [Y/n] y
        Get:1 http://us-west-1.ec2.archive.ubuntu.com/ubuntu/ trusty/universe aufs-tools amd64 1:3.2+20130722-1.1 [92.3 kB]
        Get:2 http://us-west-1.ec2.archive.ubuntu.com/ubuntu/ trusty/main liberror-perl all 0.17-1.1 [21.1 kB]
        . . .

3. 启动 Docker 集成.

        ubuntu@ip-xxx-xx-x-xxx:~$ sudo service docker start

4. 验证 Docker Engine 安装是否正确，可以执行 `docker run hello-world`.

        ubuntu@ip-xxx-xx-x-xxx:~$ sudo docker run hello-world
        ubuntu@ip-172-31-0-151:~$ sudo docker run hello-world
        Unable to find image 'hello-world:latest' locally
        latest: Pulling from library/hello-world
        b901d36b6f2f: Pull complete
        0a6ba66e537a: Pull complete
        Digest: sha256:8be990ef2aeb16dbcb9271ddfe2610fa6658d13f6dfb8bc72074cc1ca36966a7
        Status: Downloaded newer image for hello-world:latest

        Hello from Docker.
        看到这些信息说明安装正确.

        打印出上述信息Docker经历了如下过程:
        1. Docker 客户端 连接到 Docker daemon.
        2. Docker daemon 从Docker Hub中拉取 "hello-world" 镜像.
        3. Docker daemon 使用镜像创建一个新的容器，生产出你阅读的内容
        4. Docker daemon 将上述内容发送给 Docker client, 最终展示在你的终端上.

        更高阶的尝试, 你可以运行一个Ubuntu容器:
        $ docker run -it ubuntu bash

        共享镜像, 自动化工作流及更多内容请见Docker Hub:
        https://hub.docker.com

        更多实例请见:
        https://docs.docker.com/userguide/

## 继续阅读

_想更快捷的将Docker部署到云主机中?_ 可以使用 [Docker Machine](/machine/overview/) 创建云主机.

  * [使用 Docker Machine创建云主机](/machine/get-started-cloud/)

  * [Docker Machine 驱动参考](/machine/drivers/)

*  [安装 Docker Engine](../index.md)

* [Docker 用户指南](../../userguide/intro.md)
