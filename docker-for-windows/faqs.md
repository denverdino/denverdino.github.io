---
description: 常见问题
keywords: windows faqs
title: 常见问题 (FAQ)
---

>**寻找 Docker for Windows 上流行的问答?** 访问 [Docker 知识中心](http://success.docker.com/) 上的知识库文章, 常见问题解答, 各订阅级别的技术支持, 以及更多.

### 有关稳定和测试通道的问题

**Q: 如何获取稳定版或测试版的 Docker for Windows?**

A: 使用在话题[下载 Docker for Windows](index.md#download-docker-for-windows) 中给出的下载链接.

该主题还包含关于这两个通道的更多信息.

**Q:Docker for Windows 的稳定版和测试版有什么区别?**

A: Docker for Windows 提供了两种不同的下载通道:

* 稳定通道提供了一个完全成熟和测试过的更可靠的正式发行版安装包, 稳定版本的 Docker for Windows 附带了 Docker Engine 的最新发行版,
发布计划与 Docker Engine 的版本与补丁同步. 在稳定通道上, 您可以选择是否发送使用统计信息和其它数据.

* 测试通道提供了我们正在研究的新功能的安装程序, 但这些功能可能未经全面测试. 它附带了 Docker Engine 的实验性版本, 测试版应用程序更容易发生错误, 崩溃和问题, 但您可以体验新功能与实验, 并随着应用程序的发展提供意见反馈. 测试通道的发行通常比稳定版更频繁, 通常每月一次或多次. 默认情况下会发送使用情况统计信息和崩溃报告, 您无法在测试通道上停用此功能.

**Q: 我能否在稳定版与测试版的 Docker for Windows 之间切换?**

A: 是的, 您可以切换到测试版以检视新功能, 再换回到稳定版进行其它工作. 然而, **一次只能安装一个应用**.
在稳定版与测试版应用间切换可能会破坏您的开发环境, 特别是从更新的测试版切换到更旧的稳定版时.

例如, 使用较新版本的 Docker for Windows 创建的容器在​​您切换回稳定版后可能无法正常工作, 因为它们可能是利用尚未稳定的测试版功能创建的. 在创建和使用测试版容器时, 请记住这一点 (也许你是本着准备在测试空间中排除故障或重新开始的精神).

<font color="#CC3366">要在测试版和稳定版之间安全切换, 请务必保存镜像并导出所需的容器, 然后在安装另一个版本之前卸载当前版本. 下面给出了更详细的工作流程.</font><br>

请每次执行:

1. 使用 `docker save` 保存您要保留的镜像. (参见 Docker Engine 命令行参考中的[保存](/engine/reference/commandline/save.md).)

2. 使用 `docker export` 导出您要保留的容器. (参见 Docker Engine 命令行参考中的[导出](/engine/reference/commandline/export.md).)

3. 卸载当前的稳定版或测试版应用.

4. 安装另一稳定版或测试版应用.

### 我们希望什么样的反馈?

我们期待任何反馈, 我们希望了解您关于下载 / 安装过程, 启动, 可用功能, GUI, 应用可用性, 命令行集成等方面的印象. 告诉我们您遇到的问题, 喜欢的内容或您希望添加的功能.

我们对 [Docker Swarm](/engine/swarm/index.md) 中描述的新的 swarm 模式的反馈特别感兴趣, 一个好的开始的地方是[教程](/engine/swarm/swarm-tutorial/index.md).

### 如果我有问题或疑问怎么办?

您可以在[日志和故障排除](troubleshoot.md)中找到常见问题的列表.

如果在疑难解答中找不到解决方案, 请在 GitHub 上浏览或创建 [Docker for Windows 的问题](https://github.com/docker/for-win/issues). 您也可以基于诊断信息创建新问题, 要了解更多有关运行诊断程序和有关 Docker for Windows GitHub 上的问题的信息, 请参阅[诊断与反馈](index.md#diagnose-and-feedback).

[Docker for Windows 论坛](https://forums.docker.com/c/docker-for-windows)也提供了讨论帖, 您可以在那里创建讨论主题, 但我们建议使用 GitHub 问题板块而不是论坛, 以更好地跟踪和响应您的问题.

### 我可以使用 Docker for Windows 新的 swarm 模式吗?

是的! 您可以使用 Docker for Windows 测试在 Docker Engine 1.12 中加入的 [swarm 模式](/engine/swarm/index.md)的单节点功能, 包括初始化具有单个节点的集群, 创建和扩展服务. Hyper-V 上的 Docker "Moby" 将用作单个 swarm 节点. 您还可以使用 Docker for Windows 自带的 Docker Machine 来创建和实验多节点集群, 请查看[开始使用 swarm 模式](/engine/swarm/swarm-tutorial/index.md)中的教程.

### 如何连接到远程 Docker Engine API?

您可能需要向 Docker 客户端和开发工具提供 Engine API 的位置.

在 Docker for Windows上, 客户端可以通过**命名管道**连接到 Docker Engine: `npipe:////./pipe/docker_engine`, 或者通过这个 URL 的 **TCP套接字**: `http://localhost:2375`.

这将 `DOCKER_HOST` 和 `DOCKER_CERT_PATH` 环境变量设置为给定值 (对于命名管道或 TCP 套接字, 无论您用哪一个).

另请参见 [Docker Engine API](/engine/reference/api/) 和 Docker for Windows 话题[如何查找远程 API](https://forums.docker.com/t/how-to-find-the-remote-api/20988).

### 为什么 `nodemon` 不会跟踪挂载在共享驱动器上的容器中的文件更改?

目前, `inotify` 不能在 Docker for Windows 上工作. 这是一个已知的问题, 有关更多信息和临时解决方法, 请参阅在[疑难解答](troubleshoot.md)中的[inotify 在共享驱动器上不工作](troubleshoot.md#inotify-on-shared-drives-does-not-work).

### 是否支持符号链接?

Docker for Windows 支持在容器中创建的符号链接 (symlinks). 符号链接可在容器内与容器间解析, 在其他位置 (例如主机上) 创建的符号链接将无法工作.

要了解有关此限制原因的详细信息, 请参阅以下讨论:

* GitHub 上的问题: [符号链接未按预期工作](https://github.com/docker/for-win/issues/109#issuecomment-251307391)

* Docker for Windows 论坛话题: [共享驱动器上的符号链接不被支持](https://forums.docker.com/t/symlinks-on-shared-volumes-not-supported/9288)

### 如何添加自定义 CA 证书?

从 Docker for Windows 1.12.1 (2016-09-16 (稳定版) 和 Beta 26 (2016-09-14 1.12.1-beta26)) 开始, 已支持所有受信任的 CA (根 CA 或中间 CA). Docker 识别存储在受信任的根证书颁发机构或中级证书颁发机构中的证书.

Docker for Windows 将基于 Windows 证书存储创建所有用户信任的 CA 的证书包, 并将其附加到 Moby 受信任的证书中. 因此, 如果主机上的用户信任企业 SSL 证书, 它将被 Docker for Windows 信任.

要了解更多信息, 请参阅 GitHub 上的问题[允许用户添加自定义证书颁发机构](https://github.com/docker/for-win/issues/48).

### 为什么 Docker for Windows 有时会丢失网络连接 (例如 `push`/`pull` 不工作)?

网络连接在网络更改和系统睡眠期间不完全稳定, 请退出并重启 Docker 以恢复连接.

### 我可以在Docker for Windows 上使用 VirtualBox 吗?

很不幸, 当 Windows 上启用了 Hyper-V 时 VirtualBox (和其他虚拟机监控程序, 如VMWare) 将无法运行.

### 如何在 Windows Server 2016 上的 Docker 中运行 Windows 容器?

请参阅[关于 Windows 容器与 Windows Server 2016](index.md#about-windows-containers-and-windows-server-2016).

完整的教程可以在 [docker/labs](https://github.com/docker/labs) 的 [Windows 容器入门](https://github.com/docker/labs/blob/master/windows/windows-containers/README.md)中找到.

### 为什么不支持 Windows 10 家庭版?

Docker for Windows 需要 Windows Hyper-V 功能, 这在家庭版上不可用.

### 为什么需要 Windows 10?

Docker for Windows 使用 Windows Hyper-V, 虽然旧版本的 Windows 版本具有 Hyper-V, 但它们的 Hyper-V 实现缺少 Docker for Windows 需要的关键功能.

### 为什么当安装防火墙或防病毒软件时, Docker for Windows 无法启动?

某些防火墙和防病毒软件可能与 Hyper-V 和某些 Windows 10 版本 (可能是周年更新版) 不兼容, 这会影响到 Docker for Windows. 请参阅[疑难解答](troubleshoot.md)中[在安装防火墙或防病毒软件时 Docker 无法启动](troubleshoot.md#docker-fails-to-start-when-firewall-or-anti-virus-software-is-installed)的详细信息和解决方法.

### 如何卸载 Docker Toolbox?

您可能认为现在不需要 Toolbox, 因为已经有 Docker for Windows 了, 并且想要卸载它. 有关如何在 Windows 上干净的卸载 Toolbox 的详细信息, 请参阅 Toolbox Windows 主题中的[如何卸载 Toolbox](/toolbox/toolbox_install_windows.md#how-to-uninstall-toolbox).
