---
description: How swarm nodes work
keywords: docker, container, cluster, swarm mode, node
redirect_from:
- /engine/swarm/how-swarm-mode-works/
title: How nodes work
---

Docker 引擎1.12 引入了可以创建swarm模式的功能，一个或多个Docker引擎组成了swarm集群。集群由一个或多个节点组成：这些节点是运行Docker引擎1.12或更高版本的物理或虚拟机。

节点分为两类: [**管理节点**](#manager-nodes) and
[**工作节点**](#worker-nodes).

![Swarm mode cluster](../images/swarm-diagram.png)

如果您还没有准备好, 先读一下 [swarm 模式概要](../index.md) 和 [主要概念](../key-concepts.md).

## 管理节点

管理节点处理集群的管理功能：

* 维护集群状态
* 服务调度
* swarm 模式功能 [HTTP API端点](../../reference/api/index.md)

使用[Raft]（https://raft.github.io/raft.pdf）实现，管理节点维护整个集群以及在其上运行的所有服务的一致的内部状态。为了测试，可以使用单个管理节点运行群集。如果单管理节点群集中的管理节点故障，您的服务将继续运行，但您需要创建一个新群集以进行恢复。

为了利用群模式的容错功能，Docker建议使用奇数个节点以提高集群的高可用性。当有多个管理器时，您可以从管理器节点的故障中恢复，而无需停机。

* 三个管理节点的集群可以容忍一个管理节点的损失。
* 五个管理节点的集群可以容忍两个管理节点的损失。
* `N` 个管理节点的集群可以容忍`(N-1)/2`个管理节点的损失。
* Docker建议一个集群最多有七个管理器节点。

    >**注意**: 添加更多管理管理并不意味着增加可扩展性或更高的性能。实际上情况恰恰相反。

## 工作节点

工作节点也是Docker引擎的实例，其唯一目的是运行容器。 工作节点不使用raft同步分布式状态，不执行调度决策，不提供swarm模式HTTP API。

您可以创建一个只有一个管理节点的群集，但不能创建只有一个工作节点的集群。默认情况下，所有管理节点都是工作节点。在单个管理节点集群中，您可以运行诸如`docker service create`的命令，并且调度程序在本地引擎上执行所有任务。

要防止将任务过多的调度到管理节点上，请将管理节点的状态设置为`排水`模式。 调度器在`排水`模式停止节点上的任务，并会往`活动`节点上调度任务。调度器不会向配置`排水`模式的节点分配新任务。

参考[`docker node update`](../../ reference / commandline / node_update.md), 了解如何改变节点的可用性。

## 修改角色

您可以通过运行`docker node promote`来提升一个工作节点为管理节点。例如，当您将管理节点关机以进行维护时，可能需要升级工作节点为管理节点。参见[节点角色升级] (./../reference/commandline/node_promote.md)。

您还可以将管理节点降级为工作节点。请参见[节点角色降级](../../ reference / commandline / node_demote.md)。

## 更多
* 阅读关于swarm模式[services](services.md)如何工作。
* 了解[PKI](pki.md)如何在群模式下工作；
