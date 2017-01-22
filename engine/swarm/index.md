---
description: Docker Engine swarm mode overview
keywords: docker, container, cluster, swarm
title: Swarm mode overview
---
要在swarm模式下使用Docker 引擎，请从[Docker版本GitHub存储库](https://github.com/docker/docker/releases)下载安装Docker 引擎`v1.12.0`或更高版本。或者，安装最新的Docker for Mac或Docker for Windows Beta版。

Docker 引擎1.12集成了swarm模式，用于本地管理一组名为a * swarm *的Docker引擎。使用Docker CLI创建群集，将应用服务部署到群集，以及管理节点。

如果你使用的是`v1.12.0`之前的Docker版本，请参阅[Docker Swarm](/swarm)。

## 功能亮点
* **与Docker引擎集成的集群管理：**使用Docker引擎 CLI创建一组Docker引擎，您可以在其中部署应用程序服务。不需要其他编排工具来创建或管理集群。

* **分布式设计：**在部署时，Docker 集群不处理节点之间的差异，而是在运行时处理。您可以使用Docker引擎部署两种类型的节点，管理节点和工作节点。这意味着您可以从单个磁盘镜像构建整个群集。

* **可声明的服务模型：** Docker 引擎使用声明方法来让您定义应用程序堆栈中的各种服务状态。例如，您可以描述由消息队列服务、数据库后端、Web前端服务组成的应用程序。

* **可伸缩：**对于每个服务，您可以声明要运行的任务数。当您向上或向下伸缩时，swarm管理节点通过添加或删除任务来自动适应，以保持所需的状态。

* **期望的状态协调：**群体管理节点不断监视集群状态，并调整所期望状态与实际状态之间的任何差异。例如，设置一个服务运行10个容器，如果托管其中两个容器的主机崩溃，则管理节点将创建两个新容器以替换崩溃的容器。swarm管理节点将新容器分配给正在运行的可用工作节点。

* **多主机网络互联：**您可以为您的服务指定overlay网络。当swarm管理节点初始化或更新应用程序时，它会自动为overlay网络上的容器分配地址。

* **服务发现：** Swarm管理节点为swarm中的每个服务分配唯一的DNS域名，并对运行容器进行负载均衡。您可以通过嵌入在swarm中的DNS服务器查询在群中运行的每个容器。

* **负载均衡：**您可以将服务的端口暴露给外部负载均衡器。在内部，swarm允许您指定如何在节点之间分发服务容器。

* **安全：**集群中的每个节点强制执行TLS双向身份验证和加密，以确保其与所有其他节点之间的通信安全。您可以选择使用自签名根证书或自定义根CA的证书。

* **滚动更新：**在更新期间，您可以逐步向节点应用服务更新。swarm管理节点允许您对服务部署到不同节点之间的时间延迟进行配置。如果出现任何问题，您可以将任务回滚到服务的先前版本。

## 下一步
* 学习Swarm模式 [关键概念](key-concepts.md).
* 开始学[Swarm模式教程](swarm-tutorial/index.md).
* 检索Swarm模式CLI命令:
    * [swarm init](../reference/commandline/swarm_init.md)
    * [swarm join](../reference/commandline/swarm_join.md)
    * [service create](../reference/commandline/service_create.md)
    * [service inspect](../reference/commandline/service_inspect.md)
    * [service ls](../reference/commandline/service_ls.md)
    * [service rm](../reference/commandline/service_rm.md)
    * [service scale](../reference/commandline/service_scale.md)
    * [service ps](../reference/commandline/service_ps.md)
    * [service update](../reference/commandline/service_update.md)
