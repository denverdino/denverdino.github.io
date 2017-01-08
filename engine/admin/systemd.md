---
description: Controlling and configuring Docker using systemd
keywords: docker, daemon, systemd,  configuration
redirect_from:
- /engine/articles/systemd/
title: Control and configure Docker with systemd
---

许多Linux发行版使用systemd启动Docker守护程序。 这个文档显示了一些如何自定义Docker设置的示例。

## 启动 Docker守护进程

一旦安装了Docker，您将需要启动Docker守护进程。

```bash
$ sudo systemctl start docker
# or on older distributions, you may need to use
$ sudo service docker start
```

如果您希望Docker在开机时启动，您还应该：

```bash
$ sudo systemctl enable docker
# or on older distributions, you may need to use
$ sudo chkconfig docker on
```

## 自定义Docker守护进程选项

有多种方法为你的Docker守护进程配置标志和环境变量。

推荐的方法是使用systemd插件文件（如<a
target =“_ blank”
href =“https://www.freedesktop.org/software/systemd/man/systemd.unit.html”> systemd.unit </a>
文档）。 这些是名为`<something>.conf`的本地文件在`/etc/systemd/system/docker.service.d`目录。 这也可以是
`/etc/systemd/system/docker.service`，它也可以覆盖
在`/lib/systemd/system/docker.service`。

但是，如果你以前使用过一个“EnvironmentFile”的包，它通常指向`/etc/sysconfig/docker`），那么为了向后兼容，你把一个带有`.conf'` 扩展名的文件放入`/etc/systemd/systemdocker.service.d`， 目录包括以下内容：

```conf
[Service]
EnvironmentFile=-/etc/sysconfig/docker
EnvironmentFile=-/etc/sysconfig/docker-storage
EnvironmentFile=-/etc/sysconfig/docker-network
ExecStart=
ExecStart=/usr/bin/dockerd $OPTIONS \
          $DOCKER_STORAGE_OPTIONS \
          $DOCKER_NETWORK_OPTIONS \
          $BLOCK_REGISTRY \
          $INSECURE_REGISTRY
```

要检查`docker.service`是否启用可以使用`EnvironmentFile`：

```bash
$ systemctl show docker | grep EnvironmentFile

EnvironmentFile=-/etc/sysconfig/docker (ignore_errors=yes)
```

或者，找出服务文件所在的位置：

```bash
$ systemctl show --property=FragmentPath docker

FragmentPath=/usr/lib/systemd/system/docker.service

$ grep EnvironmentFile /usr/lib/systemd/system/docker.service

EnvironmentFile=-/etc/sysconfig/docker
```

您可以使用覆盖文件定制Docker守护进程选项，如使用下面的[HTTP代理示例](systemd.md＃http-proxy)。 文件位于`/usr/lib/systemd/system`或`/lib/systemd/system`包含默认选项并且不应该编辑。

### 运行时目录和存储驱动程序

您可能想要控制用于Docker镜像，容器的磁盘空间和卷通过将其移动到单独的分区。

在这个例子中，我们假设你的`docker.service`文件看起来有些东西：

```conf
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network.target

[Service]
Type=notify
# the default is not to use systemd for cgroups because the delegate issues still
# exists and systemd currently does not support the cgroup feature set required
# for containers run by docker
ExecStart=/usr/bin/dockerd
ExecReload=/bin/kill -s HUP $MAINPID
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
# Uncomment TasksMax if your systemd version supports it.
# Only systemd 226 and above support this version.
#TasksMax=infinity
TimeoutStartSec=0
# set delegate yes so that systemd does not reset the cgroups of docker containers
Delegate=yes
# kill only the docker process, not all processes in the cgroup
KillMode=process

[Install]
WantedBy=multi-user.target
```

这将允许我们通过一个插入文件（如上所述）添加额外的标志.将包含以下内容的文件放在`/etc/systemd/ system/docker.service.d`中: 

```conf
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd --graph="/mnt/docker-data" --storage-driver=overlay
```

您还可以在此文件中设置其他环境变量，例如，`HTTP_PROXY`环境变量描述如下。

要修改ExecStart配置，请指定空配置并且通过如下新配置：

```conf
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd --bip=172.17.42.1/16
```

如果无法指定空配置，Docker会报告错误，如：

```conf
docker.service has more than one ExecStart= setting, which is only allowed for Type=oneshot services. Refusing.
```

### HTTP 代理

此示例覆盖默认的“docker.service”文件

如果您在HTTP代理服务器后面，例如在公司设置中，您将需要在Docker systemd服务文件中添加此配置。

1.  为docker服务创建一个systemd插件目录：

    ```bash
    $ mkdir /etc/systemd/system/docker.service.d
    ```

2.  创建一个名为`/etc/systemd/system/docker.service.d/http-proxy.conf`的文件它添加了“HTTP_PROXY”环境变量：

    ```conf
    [Service]
    Environment="HTTP_PROXY=http://proxy.example.com:80/"
    ```

3.  如果你有内部Docker注册表，你需要联系没有代理可以通过`NO_PROXY`环境变量来指定它们：

    ```conf
    Environment="HTTP_PROXY=http://proxy.example.com:80/" "NO_PROXY=localhost,127.0.0.1,docker-registry.somecorporation.com"
    ```

4.  刷新改变:

    ```bash
    $ sudo systemctl daemon-reload
    ```

5.  验证配置是否已加载:

    ```bash
    $ systemctl show --property=Environment docker
    Environment=HTTP_PROXY=http://proxy.example.com:80/
    ```
6.  重启 Docker:

    ```bash
    $ sudo systemctl restart docker
    ```

## 手动创建systemd单元文件

当没有二进制安装包，你可能想要将Docker与systemd集成。 为此，只需安装两个单元文件
（服务和套接字）从[github存储库](https://github.com/docker/docker/tree/master/contrib/init/systemd)到`/etc/systemd/system`。
