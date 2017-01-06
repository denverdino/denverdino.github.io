---
description: Docker explained in depth
keywords: docker, introduction, documentation, about, technology,  understanding
redirect_from:
- /introduction/understanding-docker/
- /engine/userguide/basics/
- /engine/quickstart.md
- /engine/introduction/understanding-docker/
title: Docker 概述
---
 
Docker 是一个用于开发、交付和运行应用的开放平台。Docker 允许你把你的应用从基础结构上分离出来，来实现快速的软件交付。使用 Docker，你可以像管理应用那样来管理基础环境。利用 Docker 的优势来快速交付、测试和部署代码，可以显著地减少从代码编写到生产环境上线的时间。


## Docker 平台是什么？

Docker 提供了在容器（container）中打包应用并在一个宽松隔离环境中运行应用的能力。这种隔离性和安全性能够允许你在指定的宿主机上同时运行多个容器。由于容器的天生具有轻量级特性，而且没有额外的虚拟化管理程序的开销，在相同的硬件配置上，可以比运行虚拟机运行更多个容器。

Docker 提供了工具和平台来管理容器的生命周期：

* 封装应用（和支持组件）到 Docker 容器中。
* 为团队分发和交付这些容器，以便进一步的开发和测试。
* 部署这些容器到生产环境中，可以是本地的数据中心或是云端。

## 什么是 Docker Engine？

_Docker Engine_ 是一个客户端-服务端架构应用，包含如下的几个重要组件：

* 服务器端是一个长时间运行的程序，称其为守护进程（daemon process）
* REST API 是编程接口来告知守护进程做什么

* 命令行接口（CLI）客户端

![Docker Engine Components Flow](article-img/engine-components-flow.png)

通过脚本或者直接使用CLI命令，命令行接口利用REST API来控制Docker守护进程或与其交互。有许多的 Docker 应用底层就使用了 API 和 CLI。

Docker 守护进程会创建和管理 Docker __objects__ （对象），例如镜像(images)、容器(container)、网络(network) 和 数据卷 (data volomue)。

> **注：** Docker使用 Apache 2.0 许可证下的开源协议

## Docker 能做什么?

*快速地、一致地交付你的应用*

Docker 可以在开发生命周期中提升效率；使用本地容器运行应用和服务，开发者可以工作在标准化的环境中。你还可以把Docker集成到你的持续集成和持续部署（CI/CD）工作流中。

思考如下示例场景：你的开发者在本地编写代码，并通过Docker容器来和他们的同事共享工作。他们可以使用Docker把应用推送到测试环境来执行自动化和手工测试。当开发者发现问题，他们可以在本地开发环境修正后，重新部署到测试环境进行验证。当测试完成，把修正交付给客户就非常简单了，只是把更新过的镜像推送到生产环境即可。

*快速的部署和伸缩*

Docker 容器平台使得高度可移植的应用成为可能。Docker 容器可以运行在开发者本机，数据中心的物理机或虚拟机，云上，或者一个混合的环境中。

Docker 的可移植性和轻量级特性使得动态管理工作负载变得容易，可以根据业务需求近乎实时地增加或者缩小应用、服务规模。

*在相同的硬件上运行更多的应用*

Docker 是轻量和快速的。它为基于hypervisor的虚拟机提供了一个可行的，有成本效益的选择，使得你可以更加充分地利用你的计算能力来达成业务目标。这对在高密度环境和中小规模部署中，希望以较少的资源支持更多应用时，非常有用。

## Docker 的架构是什么?

Docker 采用客户机-服务器（client-server）架构。Docker *客户端* 和 Docker *daemon* 通信，来实现构建、运行和分发 Docker 容器的复杂工作。Docker *客户端* 和 Docker *daemon* 可以运行在相同的系统上。你也可以让 Docker 客户端连接远程的 Docker daemon。 Docker 客户端和 daemon 之间采用REST API，通过 UNIX sockets 或者网络进行通信。

![Docker Architecture Diagram](article-img/architecture.svg)

### Docker daemon
Docker daemon 运行在宿主机上。用户可以使用 Docker client 来和 daemon 进行交互。

### Docker client

Docker client， 也就是 `docker` 命令，是 Docker 主要的用户接口。它接受命令和配置参数来和 Docker daemon 交流。一个 Docker Client 可以和多个无关的 daemon 通信。

### Docker 内幕
理解 Docker 的内部细节，需要了解 _镜像（image）_ ， _registries_ ，和 _容器（container）_ 。

#### Docker 镜像

Docker _容器_ 是一个只读的模板，包含了用于创建 Docker 容器的指令。例如，一个镜像可以包含一个 Ubuntu 操作系统并安装了 Apache web服务器和你的应用。你可以自己从头构建、更新镜像，或者下载使用其他人创建的镜像。一个镜像可以被一个或多个镜像作为基础或者扩展。Docker 镜像采用一个称为 _Dockerfile_ 的文本文件作为描述，它具有简单和明确定义的语法。更多关于镜像的细节，请参阅 [Docker 镜像如何工作?](#docker--6).

Docker 镜像是 Docker 的 **构建** 组件。

#### Docker 容器
Docker 容器是一个 Docker 镜像的运行实例。 利用 Docker API 或 CLI，你可以运行、启动、停止、移动或者删除一个容器。当运行一个容器时，你可以提供配置元信息，比如网络信息、环境变量等。每个容器是隔离的、安全的应用平台，但是可以访问其他主机或容器上的资源，还有持久化存储或数据库。更多细节，请参阅[容器如何工作?](#section-)。

Docker 容器是 Docker 的 **运行** 组件。

#### Docker registries
Docker registry 是 Docker镜像的仓库。一个 registry 可以是公共的或者私有的，可以和 Docker daemon 或者 Docker 客户端运行在相同或者完全分离的服务器上。 更多细节，请参阅 [Docker registry如何工作?](#docker-registry-)

Docker registries 是 Docker 的 **分发** 组件。

#### Docker 服务 (Service)
Docker _服务_ 可以在一组 Docker 节点构成的 _swarm_ 集群上，运行指定数量的任务复本实例（由一个Docker镜像定义）。你可以指定并发运行的任务复本数量，swarm manager 会确保负载会平均分配在 worker 节点上。对于消费者而言，一个Docker service 代表一个单一应用。在Docker 1.12或更高版本中，Docker 引擎支持 swarm 模式。

Docker 服务是Docker的 **可伸缩性** 组件。

### Docker 镜像如何工作?
Docker 镜像是只读的模板来实例化容器。每个镜像包含一系列的层。Docker 使用
[联合文件系统](http://en.wikipedia.org/wiki/UnionFS) 来组合这些层成为一个镜像。联合文件系统使得在独立文件系统中的文件和目录，也称为分支（branches），可以透明地叠加在一起形成一个单独的一致的文件系统。

层是 Docker 如此轻量的一个原因。当你改变一个 Docker 镜像，比如将一个应用更新到新版本，一个新的层将被构建出来并仅替换掉更新过的层。其他的层保持不变。分发更新时，只需传输更新过的层。分层加速了 Docker 镜像的分发。Docker 会在运行时来决定哪些层需要被更新。

镜像是被 Dockerfile 所定义的。每个镜像都始于一个基础镜像，比如 `ubuntu` ，一个基本的 Ubuntu 镜像，或者 `fedora` ，一个基本的 Fedora 镜像。你可以使用你自己的镜像作为一个新镜像的基础，比如你有一个基础的 Apache 镜像，你可以使用它作为你所有 Web 应用镜像的基础。这个基础镜像在 Dockerfile 文件中使用 `FROM` 关键词定义。

> **Note:** [Docker Hub](https://hub.docker.com) 是一个公共的registry 来存储镜像。

Docker 镜像基于一个基础镜像，并使用 `Dockerfile` 文件中的简单的，描述性的步骤也被称为 *指令（instructions）* 进行构建。 每个指令在镜像中创建一个新的层。一些 Dockerfile 指令示例如下：

* 指明基础镜像 (`FROM`)
* 指明维护者 (`MAINTAINER`)
* 运行一个命令 (`RUN`)
* 添加一个文件或目录 (`ADD`)
* 创建环境变量 (`ENV`)
* 启动容器时执行的进程 (`CMD`)

当构建一个镜像时，Docker 会解读 `Dockerfile`，执行指令并生成镜像。

### Docker registry 如何工作?
Docker registry 保存 Docker 镜像。当你构建一个 Docker 镜像之后，你可以  *推送（push）* 它到公共的 registry 中，比如 [Docker Hub](https://hub.docker.com)
或者你防火墙内部的私有的 registry。你可以搜索已有的镜像并从 registry 拉取到主机上。

[Docker Hub](http://hub.docker.com)  是一个公共的 registry，提供了海量的已有镜像，并支持你来贡献。更多信息，请移步至
[Docker Registry](/registry/overview/) 和
[Docker Trusted Registry](/docker-trusted-registry/overview/).

[Docker store](http://store.docker.com) 支持购买和销售 Docker 镜像。你可以从软件供应商处购买一个包含了应用或服务的 Docker 镜像，并用于你的开发、测试、预发和生产环境，升级应用只需拉取一个新版本的镜像并重新部署容器。Docker Store 目前还在内测阶段。

### 容器如何工作?
容器使用宿主机的 Linux 内核，包含由镜像创建时所添加的全部文件，和创建或者启动容器时所关联的metadata组成。容器都是从镜像创建出来。镜像定义了容器的内容，比如容器的启动进程，以及一系列其他的配置细节。Docker 镜像是只读的。当 Docker 从一个镜像运行容器时，它会在镜像的顶部添加一个读/写层（使用之前提到的 UnionFS ），来让应用在其中运行。

#### 当运行一个容器时发生了什么?
当你使用 `docker run` CLI 命令或者等价的API时，Docker 客户端指挥 Docker daemon 来运行一个容器。下面示例解释了 Docker daemon 如何从 `ubuntu` Docker 镜像运行一个容器，为了保证它在前景（foreground）方式运行，采用了交互模式(`-i`)，并执行 `/bin/bash` 命令。

    $ docker run -i -t ubuntu /bin/bash


当你运行这个命令时，Docker Engine 会做如下工作：

1. **拉取 `ubuntu` 镜像:** Docker Engine 检查 `ubuntu` 镜像是否存在。如果镜像已经在本地存在，Docker Engine 使用它来创建新容器。否则 Docker Engine 会从 [Docker Hub](https://hub.docker.com) 拉取。

1. **创建一个新容器:** Docker 使用镜像来创建一个容器

1. **分配文件系统并挂载一个读/写的 _层_ :** 容器在文件系统上被创建，一个读/写层添加到镜像之上。

1. **分配网络/网桥接口:** 创建网络接口，使得 Docker 容器可以和本地主机相互通信。

1. **设置IP地址:** 从IP池中，找到一个可用地址，并挂载到容器上。

1. **执行指定进程:** 执行 `/bin/bash`。

1. **捕获并提供应用输出:** 连接标准输入、输出和错误管道。由于采用了交互式模式，使得你可以查看到你的应用执行状态。

你的容器现在已经在运行中了。你可以管理并与它交互，来使用服务和应用，最终停止并移除它。

## 底层技术
Docker 使用 [Go](https://golang.org/) 编写并得益于一些 Linux 内核特性来实现相应的功能。

### 名空间（Namespaces）
Docker使用 `名空间（namespaces）` 的技术来为 *container* 提供隔离的工作空间。当你运行一个容器时，Docker 会为容器创建一系列的 *名空间*。

这些名空间提供了一个隔离层。容器的各个方面都运行在单独的名空间中，并通过名空间中限制了它的访问。

在 Linux 上，Docker Engine 使用如下名空间：

 - **`pid` 名空间:** 进程隔离 (PID: Process ID)。
 - **`net` 名空间:** 管理网络接口 (NET: Networking)。
 - **`ipc` 名空间:** 管理 IPC 资源访问 (IPC: InterProcess Communication)。
 - **`mnt` 名空间:** 管理文件系统挂载点 (MNT: Mount)。
 - **`uts` 名空间:** 隔离内核和版本标识 (UTS: Unix Timesharing System)。

### 控制组（Control groups）
在 Linux 上， Docker Engine 还依赖于 _控制组（control groups）_ (`cgroups`)技术。一个 cgroup 可以把一个应用限制到一组指定的资源上。控制组使得 Docker Engine 可以让容器共享可用的硬件资源，并可选强制限制和约束。比如你可以限制一个容器可使用的内存量。

### 联合文件系统（Union file systems）
联合文件系统，或UnionFS，是一个可以非常轻量和快速地创建层的文件系统。Docker Engine 使用 UnionFS 来提供容器的构建组件。Docker Engine 支持多种 UnionFS，包括 AUFS，btrfs，vfs 和 DeviceMapper。

### 容器制式（container format）
Docker Engine 组合名空间（namespaces），控制组（control groups）和 UnionFS 到一个封装中，称为一个容器制式。缺省的容器制式是`libcontainer`。未来 Docker 可能会通过集成 BSD Jails 或 Solaris Zones 等技术，支持其他容器制式。

## 进一步学习
- 阅读 [安装 Docker Engine ](installation/index.md#installation).
- 亲身体验 [Docker 入门](getstarted/index.md) 教程
- 查阅[Docker Engine 用户指南](userguide/index.md)中的示例和深入话题


## 进一步学习
- Read about [Installing Docker Engine](installation/index.md#installation).
- Get hands-on experience with the [Get Started With Docker](getstarted/index.md)
    tutorial.
- Check out examples and deep dive topics in the
    [Docker Engine User Guide](userguide/index.md).


