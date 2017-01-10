---
description: 疑难解答, 日志和已知问题
keywords: windows, 疑难解答, 日志, 问题
redirect_from:
- /windows/troubleshoot/
title: 日志和疑难解答
---

本文提供了如下信息: 如何诊断和解决问题, 发送日志和与 Docker for Windows 团队联系, 使用我们的论坛与知识中心, 在 GitHub 上浏览与记录问题, 寻找已知问题的解决方法.

## Docker 知识中心

**寻找 Docker for Windows 的帮助?** 查看 [Docker 知识中心](http://success.docker.com/) 以获取各种订阅级别的知识库文章, FAQ 和技术支持.

## 提交诊断信息, 反馈与 GitHub 问题

如果遇到了在本文档中找不到解决方案的问题, 请访问 [GitHub 上的 Docker for Windows 问题](https://github.com/docker/for-win/issues) 或 [Docker for Windows 论坛](https://forums.docker.com/c/docker-for-windows), 我们可以帮助您排除日志数据故障. 请参阅 [诊断和反馈](index.md#diagnose-and-feedback) 以了解诊断以及如何在 GitHub 上创建新问题.

## 检查日志

除了使用诊断和反馈选项来提交日志之外, 您还可以自行浏览日志.

### 使用系统托盘菜单查看日志

要查看 Docker for Windows 的最新日志, 请单击系统托盘中的 `Diagnose & Feedback` 菜单项, 然后单击 `Log file` 链接. 您可以在 `AppData\Local` 文件夹中查看日志的完整历史记录.

### 使用系统托盘报告与发送问题

如果遇到问题, 但下面列出的建议故障排除过程不能解决问题, 您可以生成诊断报告. 单击系统托盘中的 `Diagnose & Feedback` 菜单项, 然后单击 `Upload diagnostic...` 链接, 这将上传诊断信息到我们的服务器, 并为您提供一个唯一的ID, 您可以使用此 ID 来在电子邮件或论坛引用您的上传.

## 故障排除

### inotify 在共享驱动器上不工作

目前, `inotify` 不能在 Docker for Windows 上工作. 例如, 当应用程序需要跨已挂载的驱动器读/写容器时, 这将变得明显. 这是团队正在努力解决的已知问题. 以下是临时解决方案与该问题的链接.

* **nodemon 与 Node.js 的解决方案** - 如果您在 `Node.js` 上使用 [nodemon](https://github.com/remy/nodemon) with  `Node.js`, 请尝试此处描述的回退轮询模式: [nodemon isn't restarting node
applications](https://github.com/remy/nodemon#application-isnt-restarting)

* **GitHub 上的 Docker for Windows 问题** - 查看问题 [Inotify on shared drives does not work](https://github.com/docker/for-win/issues/56#issuecomment-242135705)

### Linux 容器以及任何在 `C:\Users` 目录之外的项目挂载卷时需要共享驱动器

如果挂载卷时发生应用程序文件无法找到, 卷拒绝挂载或服务无法启动 (例如使用 [Docker Compose](/compose/gettingstarted.md) 时) 的运行时错误, 您可能需要启用](index.md#shared-drives).

Linux 容器在挂载卷时需要共享驱动器. 请转到 <img src="images/whale-x.png"> --> **Settings** --> **Shared Drives** 来共享包含了 Dockerfile 与卷的驱动器.

### 验证域用户是否具有共享驱动器 (卷) 的权限

>**提示:** 共享驱动器仅在 [Linux 容器](index.md#switch-between-windows-and-linux-containers-beta-feature)挂载卷时需要, 而 Windows 容器不需要.

访问共享驱动器的权限和用于设置共享驱动器的用户名与密码有关. (请参阅[共享驱动器](index.md#shared-drives).)
如果以与用于设置共享驱动器不同的用户名运行 `docker` 命令和任务, 您的容器将无权访问已挂载的卷, 卷将显示为空.

此问题的解决方法是切换到域用户帐户并重置共享驱动器上的凭据.

下面的示例说明了如何解决这个问题, 假定一种场景: 您作为本地用户 (而非域用户) 共享了 `C` 驱动器, 本地用户是 `samstevens`, 域用户是 `merlin`.

1. 确保您以 Windows 域用户身份登录 (对于本例, 使用 `merlin`).

2. 运行 `net share c` 查看 `<host>\<username>, FULL` 的用户权限.

		PS C:\WINDOWS\system32> net share c
		Share name        C
		Path              C:\
		Remark
		Maximum users     No limit
		Users             SAMSTEVENS
		Caching           Caching disabled
		Permission        windowsbox\samstevens, FULL

3. 运行以下命令来移除共享.

		net share c /delete

4. 通过[共享驱动器对话框](index.md#shared-drives)重新共享驱动器, 并提供 Windows 域用户账户凭据.

5. 重新运行 `net share c`.

		PS C:\WINDOWS\system32> net share c
		Share name        C
		Path              C:\
		Remark
		Maximum users     No limit
		Users             MERLIN
		Caching           Caching disabled
		Permission        windowsbox\merlin, FULL

另请参见 GitHub 上的相关问题 [Mounted volumes are empty in the container](https://github.com/docker/for-win/issues/25).

### 本地安全策略可能阻止共享驱动器并导致登录错误

您需要权限才能挂载共享驱动器以使用 Docker for Windows 的[共享驱动器](index.md#shared-drives)功能.

如果被本地策略阻止, 在 Docker 上尝试启用共享驱动器时您将收到错误, 此问题 Docker 无法解决, 您需要提供权限才能使用此功能.

这里是示例错误消息的片段:

```
Logon failure: the user has not been granted the requested logon type at
this computer.

[19:53:26.900][SambaShare     ][Error  ] Unable to mount C drive: mount
error(5): I/O error Refer to the mount.cifs(8) manual page (e.g. man mount.cifs)
mount: mounting //10.0.75.1/C on /c failed: Invalid argument
```

另请参见 <a href="https://github.com/docker/for-win/issues/98">Docker for Windows 问题 #98</a>.

### 了解符号链接的限制

符号链接将在容器内和容器间工作, 但在容器外部 (例如在主机上) 创建的符号链接将不起作用. 要了解更多信息, 请参阅 FAQ 的[是否支持符号链接?](faqs.md#are-symlinks-supported)

### 避免未期望的语法错误, 对容器中的文件请使用 Unix 风格的行尾序列

任何要在容器中运行的文件必须使用 Unix 样式 `\n` 的行结尾, 这包括用于编译的命令行中引用的文件, 以及在 Docker 文件的 RUN 命令引用的文件.

Docker 容器和 `docker build` 在 Unix 环境中运行, 因此容器中的文件必须使用 Unix 样式行结尾: `\n`, 而_不是_ Windows 样式: `\r\n`.
请注意, 在使用 Windows 工具创建诸如 shell 脚本等文件时, 行尾默认值可能是 Windows 样式.
这些命令最终会传递到基于 Unix 的容器中的 Unix 命令 (例如, shell 脚本传递到 `/bin/sh`), 如果使用 Windows 样式行尾, `docker run` 将失败并出现语法错误.

有关此问题和解决方案的示例, 请参阅 GitHub 上的问题: [Docker RUN 无法执行 shell 脚本](https://github.com/docker/docker/issues/24388).

### 在 Beta 18 升级后重新创建或更新容器

Docker 1.12.0 RC3 版本引入了从 RC2 到 RC3 的向后不兼容更改. (有关详情, 请参阅 https://github.com/docker/docker/issues/24343#issuecomment-230623542.)

当您尝试启动使用早于 Beta 18 的 Docker for Windows 创建的容器时, 可能会收到以下错误.

    Error response from daemon: Unknown runtime specified default

此问题可通过[重新创建](troubleshoot.md#recreate-your-containers)或[更新](troubleshoot.md#update-your-containers)容器来解决.

如果您收到上面显示的错误消息, 我们建议您重新创建容器.

#### 重新创建

要重新创建容器, 请使用 Docker Compose.

    docker-compose down && docker-compose up

#### 更新容器

要修复已有容器, 请按以下步骤操作:

1.  运行下面的命令.

    ```bash
    $ docker run --rm -v /var/lib/docker:/docker cpuguy83/docker112rc3-runtimefix:rc3

    Unable to find image 'cpuguy83/docker112rc3-runtimefix:rc3' locally
    rc3: Pulling from cpuguy83/docker112rc3-runtimefix
    91e7f9981d55: Pull complete
    Digest: sha256:96abed3f7a7a574774400ff20c6808aac37d37d787d1164d332675392675005c
    Status: Downloaded newer image for cpuguy83/docker112rc3-runtimefix:rc3
    proccessed 1648f773f92e8a4aad508a45088ca9137c3103457b48be1afb3fd8b4369e5140
    skipping container '433ba7ead89ba645efe9b5fff578e674aabba95d6dcb3910c9ad7f1a5c6b4538': already fixed
    proccessed 43df7f2ac8fc912046dfc48cf5d599018af8f60fee50eb7b09c1e10147758f06
    proccessed 65204cfa00b1b6679536c6ac72cdde1dbb43049af208973030b6d91356166958
    proccessed 66a72622e306450fd07f2b3a833355379884b7a6165b7527c10390c36536d82d
    proccessed 9d196e78390eeb44d3b354d24e25225d045f33f1666243466b3ed42fe670245c
    proccessed b9a0ecfe2ed9d561463251aa90fd1442299bcd9ea191a17055b01c6a00533b05
    proccessed c129a775c3fa3b6337e13b50aea84e4977c1774994be1f50ff13cbe60de9ac76
    proccessed dea73dc21126434f14c58b83140bf6470aa67e622daa85603a13bc48af7f8b04
    proccessed dfa8f9278642ab0f3e82ee8e4ad029587aafef9571ff50190e83757c03b4216c
    proccessed ee5bf706b6600a46e5d26327b13c3c1c5f7b261313438d47318702ff6ed8b30b
    ```

2. 退出 Docker.

3. 启动 Docker.

	> **注意:** 在尝试启动容器之前, 请务必退出并重新启动 Docker for Windows.

4. 尝试再次启动容器:

    ```bash
    $ docker start old-container
    old-container
    ```

### Hyper-V

Docker for Windows 需要安装和启用 Hyper-V 和用于 Windows Powershell 的 Hyper-V 模块. Docker for Windows 安装程序将帮助您启用.

参见这些[说明](https://msdn.microsoft.com/en-us/virtualization/hyperv_on_windows/quick_start/walkthrough_install)来手动安装 Hyper-V.
安装后*需要*重新启动. 如果安装 Hyper-V 后没有重启, Docker for Windows 将不能正常工作.
在某些系统上, 需要在 BIOS 中启用虚拟化, 操作随制造商的不同而不同, 但通常 BIOS 中的选项被称为 `Virtualization Technology (VTx)` 或类似名称.

### 必须启用虚拟化

除了 [Hyper-V](#hyper-v) 之外, 还必须启用虚拟化.

如果在手动卸载了 Hyper-V 或禁用了虚拟化, Docker for Windows 将无法启动.

在任务管理器中验证虚拟化是否开启:

![Task Manager](images/win-virtualization-enabled.png)

另请参阅用户报告的问题 [无法在 Windows 10 企业版中运行 Docker for Windows](https://github.com/docker/for-win/issues/74).

### Windows 容器和 Windows Server 2016

如果您对如何在 Windows Server 2016 或 Windows 10 上安装和运行 Windows 容器有任何疑问, 请参阅[关于 Windows 容器和 Windows Server 2016](index.md#about-windows-containers-and-windows-server-2016).

完整的教程可以在 [docker/labs](https://github.com/docker/labs) 的 [Windows 容器入门](https://github.com/docker/labs/blob/master/windows/windows-containers/README.md) 中找到.

您可以安装本机 Windows 二进制文件, 它允许您在没有 Docker for Windows 时开发和运行 Windows 容器, 但如果以这种方式安装 Docker, 将无法开发或运行 Linux 容器.
在本机 Docker 守护进程上运行 Linux 容器将发生错误:

   ```no-highlight
   C:\Program Files\Docker\docker.exe:
   image operating system "linux" cannot be used on this platform.
   See 'C:\Program Files\Docker\docker.exe run --help'.
   ```

### 网络问题

有些用户报告了在 Docker for Windows 稳定版本上连接 Docker Hub 的问题. (参见 GitHub 问题 [22567](https://github.com/docker/docker/issues/22567).)

这是一个示例命令与错误消息:

	PS C:\WINDOWS\system32> docker run hello-world
	Unable to find image 'hello-world:latest' locally
	Pulling repository docker.io/library/hello-world
	C:\Program Files\Docker\Docker\Resources\bin\docker.exe: Error while pulling image: Get https://index.docker.io/v1/repositories/library/hello-world/images: dial tcp: lookup index.docker.io on 10.0.75.1:53: no such host.
	See 'C:\Program Files\Docker\Docker\Resources\bin\docker.exe run --help'.

解决本问题的直接方法是, 重置 DNS 服务器以使用 Google DNS 固定地址: `8.8.8.8`.
您可以在 **Settings** -> **Network** 对话框中进行配置, 如[网络](index.md#network)主题中所述.
当您应用此设置时, Docker 会自动重新启动, 这可能需要一些时间.

我们目前正在跟进此问题.

#### Beta 10 之前版本的网络问题

Docker for Windows Beta 10及更高版本修复了有关网络设置的一些问题. 如果您仍然遇到网络问题, 可能与以前版本的
Docker for Windows 有关, 在这种情况下, 请退出 Docker for Windows 并执行以下步骤:

##### 1.删​​除所有 DockerNAT 虚拟交换机

您可能有多个叫做 `DockerNAT` 的内部虚拟交换机, 可以通过 `Hyper-V 管理器` 子菜单的虚拟交换机管理器或通过在提
升的 Powershell (作为管理员身份运行) 中键入 `Get-VMSwitch` 来检视所有的虚拟交换机. 要删除 `DockerNAT` 名
字的虚拟交换机, 可以通过 `虚拟交换机管理器` 或 powershell 命令 `Remove-VMSwitch` 来实现.

##### 2. 删除残留的 IP 地址

您的系统上可能存在残留的 IP 地址, 它们应该在删除相关联的虚拟交换机时被删除, 但有时会删除失败, 可以在提升的
Powershell 提示符中使用 `Remove-NetIPAddress 10.0.75.1` 来移除它们.

##### 3. 删除过时的 NAT 配置

您的系统上可能存在过时的 NAT 配置, 可通过在提升的 Powershell 提示符中使用 `Remove-NetNat DockerNAT` 来移除它们.

##### 4. 删除过时的网络适配器

您的系统上可能存在过时的网络适配器, 可通过在提升的 Powershell 提示符中执行如下命令来移除它们:

    $vmNetAdapter = Get-VMNetworkAdapter -ManagementOS -SwitchName DockerNAT
    Get-NetAdapter "vEthernet (DockerNAT)" | ? { $_.DeviceID -ne $vmNetAdapter.DeviceID } | Disable-NetAdapter -Confirm:$False -PassThru | Rename-NetAdapter -NewName "Broken Docker Adapter"

接着您可以通过 `devmgmt.msc` (即设备管理器) 来手动删除它们, 您应该可以在 "网络适配器" 节中看到禁用的 Hyper-V 虚拟以太网适配器, 右键单击并选择 **卸载** 可以删除适配器.

### NAT/IP 配置

默认情况下, Docker for Windows 使用内部网络前缀 `10.0.75.0/24`, 如果这与您的正常网络设置冲突, 您可以从 **Settings** 菜单更改前缀, 请参阅[设置](index.md#docker-settings)下的[网络](index.md#network)主题.

#### Beta 15 之前版本的 NAT/IP 配置问题

从 Beta 15 版本开始, Docker for Windows 不再使用具有 NAT 配置的交换机, 下面的注释仅限于较旧的测试版本.

从 Beta 14 版本开始, Docker for Windows 的网络可以通过 UI 进行配置, 请参阅[设置](index.md#docker-settings)下的[网络](index.md#network)主题.

默认情况下, Docker for Windows 使用 NAT 前缀配置为 `10.0.75.0/24` 的内部 Hyper-V 交换机, 您可以通过 **Settings** 菜单更改使用的前缀以及 DNS 服务器, 如网络主题中所述.

如果您有额外的 Hyper-V 虚拟机, 且它们配置了自己的 NAT 前缀, 由于受到 Windows NAT 实现的限制, 前缀应仔细管理. 具体来说, Windows 目前只允许一个内部 NAT 前缀, 如果您的其它虚拟机需要的额外的前缀, 则可以创建一个更大的 NAT 前缀.

要创建较大的 NAT 前缀, 请执行以下操作:

1. 停止 Docker for Windows 并使用 `Remove-NetNAT` 删除所有 NAT 前缀.

2. 创建一个新的较短的 NAT 前缀, 其涵盖了 Docker for Windows 的 NAT 前缀和额外的 NAT 前缀, 例如:

        New-NetNat -Name DockerNAT -InternalIPInterfaceAddressPrefix 10.0.0.0/16

  下次 Docker for Windows 启动时, 它将使用新的更大范围的前缀.

或者您也可以使用不同的 NAT 名称与 NAT 前缀, 并相应的通过 `Settings` 面板调整 Docker for Windows 使用的 NAT 前缀.

>**注意**: 您还需要调整现有的虚拟机以使用新的 NAT 前缀中的 IP 地址.

### 主机文件系统共享

Docker for Windows 使用的 Linux 虚拟机 使用 SMB/CIFS 来挂载主机文件系统, 为了使用此功能, 您必须通过 `Settings` 菜单显式启用它. 系统将提示您输入用户名和密码.

不幸的是, 此设置不支持包含 Unicode 字符的密码, 因此您的密码必须为 8 位的 ASCII字符.

该设置也不支持空密码, 因此如果要使用主机文件系统共享功能, 您必须设置密码. Beta 11 和更新版本的 Docker for Windows 将显示警告, 但早期版本不会.

注意, Beta 11 之前版本的 Docker for Windows 也不支持用户名和密码中包含空格, 但这已经在 Beta 11 版本中修复.

请确保在 `控制面板\网络和 Internet\网络和共享中心\更改高级共享设置` 中启用了 "文件和打印机共享".

![共享设置](images/win-file-and-printer-sharing.png)

## 解决方法

### `inotify` 目前不能在 Docker for Windows 中工作

如果您在 `Node.js` 上使用 `nodemon`, 临时解决方法是尝试此处描述的回退轮询模式: [nodemon isn't restarting node applications](https://github.com/remy/nodemon#application-isnt-restarting). See also this issue on GitHub [Inotify on shared drives does not work](https://github.com/docker/for-win/issues/56#issuecomment-242135705).

### 重启

重新启动 PC 以停止 / 丢弃从先前安装版本运行的守护进程的任何痕迹.

### 取消设置 `DOCKER_HOST`

`DOCKER_HOST` 不需要被设置, 因为它可能指向了另一个 Docker (例如 VirtualBox), 所以应该取消设置.
如果您使用 bash, `unset ${!DOCKER_*}` 将取消设置现有的 `DOCKER` 环境变量, 对于其它 shell, 单独取消设置每个环境变量.

### 确保 Docker 正在运行 webserver 示例

对于 `hello-world-nginx` 和其它示例, Docker for Windows 必须运行才能访问 `http://localhost/`. 请确保
Docker 的鲸鱼图标在任务栏托盘中显示, 并且您是在连接到 Docker for Windows 引擎的 shell 中运行 Docker 命令
(而非来自 Toolbox 的引擎), 否则可能使用 docker 启动了 webserver 容器却得到 "网页不可用" 的错误. 有关区分这
两个环境的更多信息, 请参阅[入门](index.md)中的 "运行 Docker for Windows 与 Docker Toolbox".

### 如何解决`端口已分配`的错误

如果您得到类似 `Bind for 0.0.0.0:8080 failed: port is already allocated` 或
`listen tcp:0.0.0.0:8080: bind: address is already in use` 的错误...

这些错误通常是由 Windows 上使用这些端口的其它软件造成的, 要定位这个软件, 请使用 `resmon.exe` GUI, 然后单击
"网络", "侦听端口", 或在 powershell 中使用 `netstat -aon | find /i "listening "` 来发现当前使用端口的进
程的 PID (PID 是最右边列中的数字). 您可以关闭其它进程, 或在 docker 应用程序中使用不同的端口.

### 当安装防火墙或防病毒软件时, Docker 无法启动

**某些防火墙和防病毒软件可能与 Microsoft Windows 10 不兼容** (例如, Windows 10 周年更新). 这种冲突通常在 Windows 更新或全新安装防火墙后发生, 并且表现为来自 Docker 守护进程和 **Docker for Windows 启动失败**的错误响应. Comodo 防火墙就是该问题的一个例子, 但用户报告称该软件已更新为可与这些 Windows 10 的版本一起工作.

请参阅 Comodo 论坛主题 [Comodo Firewall conflict with
Hyper-V](https://forums.comodo.com/bug-reports-cis/comodo-firewall-began-conflict-with-hyperv-t116351.0.html) 与 [Windows 10 Anniversary build doesn't allow Comodo drivers to be
installed](https://forums.comodo.com/install-setup-configuration-help-cis/windows-10-aniversary-build-doesnt-allow-comodo-drivers-to-be-installed-t116322.0.html). 有 Docker for Windows 用户创建的问题特别描述了与 Docker 相关的问题: [Docker fails to start on Windows
10](https://github.com/docker/for-win/issues/27).

作为临时的应对方法, 请卸载防火墙或防病毒软件, 或浏览论坛上建议的其它解决方法.
