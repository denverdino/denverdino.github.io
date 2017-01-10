---
description: Instructions for installing Docker on Red Hat Enterprise Linux.
keywords: Docker, Docker documentation, requirements, linux,  rhel
redirect_from:
- /engine/installation/rhel/
- /installation/rhel/
title: Install Docker on Red Hat Enterprise Linux
---

Red Hat Enterprise Linux 7支持Docker。该安装说明适用发行版以及Docker的安装机制，
请确定你获取了最新的Docker版本。如果你想使用Red Hat工具包安装，请查阅你的Red Hat发行文档。

## 先决条件

Docker 需要运行在Linux内核版本为3.10或更高的64位操作系统上。

检查当前系统的内核版本，打开命令行终端并且执行 `uname -r`命令来显示你的系统的内核版本:


```bash
$ uname -r
3.10.0-229.el7.x86_64
```

最后，强烈建更新你的系统。确保你的系统已经完全打了修复任何潜在内核bug的补丁。
最新的内核包已经包含了被提交的内核bug。

## 安装 Docker Engine

目前有两种方式来安装Docker Engine.  你可以 [通过 `yum`包管理方式安装](#install-with-yum). 
另外，也可以使用 `curl` 命令 [访问`get.docker.com`网站](#install-with-the-script). 
这第二种方法运行一个安装脚本也通过`yum`的包管理器安装。

### 通过yum安装

1. 使用 `sudo` 或者 `root`权限的用户登陆系统.

2. 确保你已安装的软件包是最新的.

    ```bash
    $ sudo yum update
    ```

3. 添加 `yum` 源.

    ```bash
    $ sudo tee /etc/yum.repos.d/docker.repo <<-'EOF'
    [dockerrepo]
    name=Docker Repository
    baseurl=https://yum.dockerproject.org/repo/main/centos/7/
    enabled=1
    gpgcheck=1
    gpgkey=https://yum.dockerproject.org/gpg
    EOF
    ```

4. 安装Docker.

    ```bash
    $ sudo yum install docker-engine
    ```

5. 设置系统服务.

    ```bash
    $ sudo systemctl enable docker.service
    ```

6. 启动Docker.

    ```bash
    $ sudo systemctl start docker
    ```

7. 运行一个测试镜像在容器中，来验证 `docker` 是否成功安装.

        $ sudo docker run --rm hello-world

        Unable to find image 'hello-world:latest' locally
        latest: Pulling from library/hello-world
        c04b14da8d14: Pull complete
        Digest: sha256:0256e8a36e2070f7bf2d0b0763dbabdd67798512411de4cdcf9431a1feb60fd9
        Status: Downloaded newer image for hello-world:latest

        Hello from Docker!
        This message shows that your installation appears to be working correctly.

        To generate this message, Docker took the following steps:
         1. The Docker client contacted the Docker daemon.
         2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
         3. The Docker daemon created a new container from that image which runs the
            executable that produces the output you are currently reading.
         4. The Docker daemon streamed that output to the Docker client, which sent it
            to your terminal.

        To try something more ambitious, you can run an Ubuntu container with:
         $ docker run -it ubuntu bash

        Share images, automate workflows, and more with a free Docker Hub account:
         https://hub.docker.com

        For more examples and ideas, visit:
         https://docs.docker.com/engine/userguide/

如果你想要添加一个 HTTP 代理，为 Docker 运行文件设置不同的目录或分区，又或者定制一些其它的功能，请阅读我们的系统文章，
了解[如何定制Docker进程](../../admin/systemd.md).

### 通过脚本安装

1. 使用 `sudo` 或者 `root`权限的用户登陆系统.

2. 确保你已安装的软件包是最新的.

    ```bash
    $ sudo yum update
    ```

3. 运行Docker 安装脚本.

    ```bash
    $ curl -fsSL https://get.docker.com/ | sh
    ```

    该脚本添加 `docker.repo` 源，并且安装Docker.

4. 设置系统服务.

    ```bash
    $ sudo systemctl enable docker.service
    ```

5. 启动Docker.

    ```bash
    $ sudo systemctl start docker
    ```

6. 运行一个测试镜像在容器中，来验证 `docker` 是否成功安装.

    ```bash
    $ sudo docker run hello-world
    ```

如果你想要添加一个 HTTP 代理，为 Docker 运行文件设置不同的目录或分区，又或者定制一些其它的功能，请阅读我们的系统文章，
了解[如何定制Docker进程](../../admin/systemd.md).

## 创建docker组

`docker`守护进程绑定一个Unix socket而不是TCP的端口。默认情况下，Unix socket是属于用户`root`的，其他
用户可以通过`sudo`来访问。因此，`docker` 守护进程需要以`root`身份运行。

为了避免在你使用`docker`命令的时候需要使用`sudo`，创建一个`docker`的Unix用户组并且添加用户到组里。
当`docker`守护进程启动时，确保`docker`组对Unix socket有正确的读/写权限。

>**警告**: `docker` 用户组等同于 `root` 用户; 有关更多关于对你系统安全方面的影响，
> 请参考[*Docker Daemon Attack
>Surface*](../../security/security.md#docker-daemon-attack-surface) .

创建 `docker` 用户组并且添加用户:

1.  使用 `sudo` 或者 `root`权限的用户登陆系统.

2.  创建 `docker` 用户组.

    ```bash
    $ sudo groupadd docker
    ```

3. 添加用户到 `docker` 用户组.

    ```bash
    $ sudo usermod -aG docker your_username`
    ```

4. 重新登陆.

    这样确保用户运行在正确的权限下.

5. 通过直接运行 `docker` 而不是 `sudo`来验证用户在正常的组里.

    ```bash
    $ docker run hello-world
    ```

## 设置开机启动Docker

配置Docker守护进程开机启动：

```bash
$ sudo systemctl enable docker
```

## 卸载

你可以通过`yum`方式卸载Docker.

1. 列出已安装的Docker软件包.

    ```bash
    $ yum list installed | grep docker

    docker-engine.x86_64     1.7.1-0.1.el7@/docker-engine-1.7.1-0.1.el7.x86_64
    ```

2. 移除软件包.

    ```bash
    $ sudo yum -y remove docker-engine.x86_64
    ```

	该命令并不会移除镜像，容器，数据卷以及用户创建的配置文件等.

3. 运行一下命令来删除全部的镜像，容器，以及数据卷:

    ```bash
    $ rm -rf /var/lib/docker
    ```

4. 定位并删除用户创建的配置文件.
