---
description: Manager administration guide
keywords: docker, container, swarm, manager, raft
redirect_from:
- /engine/swarm/manager-administration-guide/
title: Administer and maintain a swarm of Docker Engines
---

当你通过swarm运行一组docker engine，管理节点作为关键组件来管理整个swarm并保存swarm的状态。为了部署和维护swarm集群，理解管理节点的关键特性是非常重要的。

这篇文章覆盖以下Swarm管理任务:

* [使用静态IP作为管理节点的广播地址](#use-a-static-ip-for-manager-node-advertise-address)
* [添加管理节点提高容灾能力](#add-manager-nodes-for-fault-tolerance)
* [分散式管理节点](#distribute-manager-nodes)
* [管理节点只有管理功能](#run-manager-only-nodes)
* [备份Swarm状态](#back-up-the-swarm-state)
* [监控Swarm健康状态](#monitor-swarm-health)
* [管理节点故障排除](#troubleshoot-a-manager-node)
* [强制移除节点](#force-remove-a-node)
* [灾难恢复](#recover-from-disaster)
* [强制swarm重新调度](#forcing-the-swarm-to-rebalance)

参考[节点如何工作](how-swarm-mode-works/nodes.md)，查看Docker Swarm 模式的概要介绍和管理节点与工作节点的不同


## 在Swarm中运行管理节点
Swarm 管理节点使用[Raft一致性算法](raft.md)管理Swarm状态信息。为了理解Swarm工作原理，你需要了解一些Raft的基本概念。

管理节点的数量没有限制，决定管理节点数量取决于性能和容错能力之间的群和权衡；向Swarm中添加越多的管理节点，swarm的容错能力越强。然而由于每个管理节点都要同步swarm状态，则过多的管理节点会降低集群的性能，也意味着更多的网络流量。

Raft需要在多个管理节点中选一个领导者（仲裁者），用来确定Swarm的状态更新，同时也会决定节点的添加和删除。成员操作受限于状态响应。


## 使用一个静态IP作为管理节点的通告地址

当初始化一个swarm集群时，通过配置 `--advertise-addr` 参数来向其他管理节点通告swarm地址。 

更多信息请参考 [Swarm模式下运行docker engine](swarm-mode.md#configure-the-advertise-address). 由于管理节点作为整个架构中的稳定设施，所以需要适应一个固定的 *IP地址* 以防在机器重启时swarm产生不稳定情况。

如果整个swarm重启，并且每个管理节点频繁的获得新的IP地址，工作节点将不能稳定的链接到管理节点，如果工作节点使用老的IP地址去连接管理节点，swarm可能会被挂起而不能工作。

对于工作节点来说动态IP是可以使用的。

## 添加管理节点增强容器能力

Swarm集群中应该配置奇数个管理节点。维持奇数个管理节点可以在网络隔离时，管理节点的首领保持处理请求的能力。当网络被隔离时，不能保证只会维护一个管理节点领导者。

| Swarm 大小 |  管理节点  |  容错能力  |
|:------------:|:----------:|:-----------------:|
|      1       |     1      |         0         |
|      2       |     2      |         0         |
|    **3**     |     2      |       **1**       |
|      4       |     3      |         1         |
|    **5**     |     3      |       **2**       |
|      6       |     4      |         2         |
|    **7**     |     4      |       **3**       |
|      8       |     5      |         3         |
|    **9**     |     5      |       **4**       |

例如：一个swarm中有 *5个节点*，如果丢失了 *3个*，将不会有领导者。在恢复不可用管理节点或者使用灾难恢复命令恢复swarm前，不允许向swarm中添加或者删除节点。参考 [灾难恢复](#recover-from-disaster).

把Swarm集群伸缩到一个节点是被允许的，但是swarm自己不能删除最后一个节点。这样才能保证可以访问swarm，并处理swarm请求；集群伸缩到一个节点是不安全的、也是不推荐的。如果最后一个节点被从swarm中删除，swarm会不可用，除非重启这个节点或者通过`--force-new-cluster`标签重启swarm。

可以通过`docker swarm` 和`docker node`管理swarm节点。参考[向swarm中添加节点](join-nodes.md)，了解如何添加工作节点，升级一个工作节点为管理节点。

## 分布式管理节点

使用奇数个管理节点，要多注意数据中心拓扑的地域分布。为了提高容错能力，管理节点分布上要最少跨越3个可用区。这样即使一个可用区出现了问题，swarm会重新选择一个领导者来处理请求，并重新均衡工作流。

| Swarm 管理节点 |  分配情况 (3个可用区) |
|:-------------------:|:--------------------------------------:|
| 3                   |                  1-1-1                 |
| 5                   |                  2-2-1                 |
| 7                   |                  3-2-2                 |
| 9                   |                  3-3-3                 |

## 只有管理节点的集群
管理节点默认也是工作节点。这也意味着调度器会把工作任务分配给管理节点。在管理节点上执行一些小的、非关键的任务风险是比较低的，而且可以通过**限制** *CPU*、*内存* 的使用量来增加安全性。

由于管理节点通过使用raft一致性算法在节点之间保持数据一致性，这会使管理节点对资源缺乏非常敏感。最好是把管理节点独立出来，以免影响swarm的心跳监测或者选主。

为了避免与管理节点混淆，可以通过对管理节点进行排水操作，以使其不会被用作工作节点：

```bash
docker node update --availability drain <NODE>
```

当对一个节点执行排水操作时，调度器会把在此节点执行的认为重新调度到其他工作节点。同时也不会再给这个节点分配新的任务。

## 备份swarm状态信息

管理节点在以下目录中保存swarm状态和日志：

```bash
/var/lib/docker/swarm/raft
```

经常的备份`raft`目录，以备[灾难恢复](#recover-from-disaster)的时候使用。可以把集群中一个管理节点的`raft`目录转存到一个新的swarm集群。


## 监控swarm健康
可以通过JSON格式的Docker `nodes` API（`nodes`）来监控管理节点的健康状态。参考[节点API文档](../reference/api/docker_remote_api_v1.24.md#36-nodes)了解更多信息；
在命令行下，执行`docker node inspect <id-node>` 来查看节点信息。例如：检查管理节点是否可以到达：

```bash
{% raw %}
docker node inspect manager1 --format "{{ .ManagerStatus.Reachability }}"
reachable
{% endraw %}
```

检查工作节点的状态：

```bash
{% raw %}
docker node inspect manager1 --format "{{ .Status.State }}"
ready
{% endraw %}
```
通过上面的命令，可以发现`manager1`即符合`可达的`的条件，又符合`准备好的`的条件，所以manager1既是管理节点，也是工作节点；

一个 `不可达`的健康状态意味着其他管理节点对这个管理节点是不可达的。在这个情况需要额外操作来恢复不可达节点：

- 重启daemon，观察管理节点是否可达；
- 重启机器；
- 如果上面2步不起作用，则需要添加其他管理节点或者把一个工作节点提升为管理节点。同时需要执行`docker node demote <NODE>`和`docker node rm <id-node>`来清理失败的节点。

可以通过`docker node ls`命令对整个swarm集群的健康状态进行查询。

```bash

docker node ls
ID                           HOSTNAME  MEMBERSHIP  STATUS  AVAILABILITY  MANAGER STATUS
1mhtdwhvsgr3c26xxbnzdc3yp    node05    Accepted    Ready   Active
516pacagkqp2xc3fk9t1dhjor    node02    Accepted    Ready   Active        Reachable
9ifojw8of78kkusuc4a6c23fx *  node01    Accepted    Ready   Active        Leader
ax11wdpwrrb6db3mfjydscgk7    node04    Accepted    Ready   Active
bb1nrq2cswhtbg4mrsqnlx1ck    node03    Accepted    Ready   Active        Reachable
di9wxgz8dtuh9d2hn089ecqkf    node06    Accepted    Ready   Active
```

## 管理节点错误排查
不可以从其他节点拷贝`raft`目录后重启管理节点，这个目录的数据是对节点ID唯一匹配的。一个节点只能使用一个节点ID加入swarm集群一次，所以节点ID是全局唯一的。

向集群中重新添加管理节点：

1.	把管理节点属性降级为工作节点，命令：`docker node demote <NODE>`
2.	将节点从Swarm中删除，命令：`docker node rm <NODE>`
3.	以新节点的形式重新加入集群，命令：`docker swarm join`

更多关于如何添加管理节点信息，请参考[集群中添加节点](join-nodes.md).

## 强制删除节点
多数情况下在通过`docker node rm`命令删除节点之前应该先停止节点。如果一个节点变的不可达、无响应或者响应很慢时，你可以通过添加`--force`标签强制删除节点，而不必先去停止他。例如：`node9`变的很慢时：

<!-- bash hint breaks block quote -->
```
$ docker node rm node9

Error response from daemon: rpc error: code = 9 desc = node node9 is not down and can't be removed

$ docker node rm --force node9

Node node9 removed from swarm
```

在强制删除节点前，首先需要把节点降级为工作节点。在降级或者删除节点时，确保一直有奇数个管理节点。

## 灾难恢复
swarm对错误是有一定弹性的，可以对任何数量的临时失败节点进行恢复（通过重启机器或重启docker守护进程）

集群中有`N`个管理节点时，活跃管理节点数量要超过所有管理节点的一半（`(N/2)+1`），以保证swarm能处理请求、维持可用。这意味着swarm集群允许`(N-1)/2`个故障，如果超过这个数量则无法处理集群请求。

即使您遵循本指南，也可能会丢失仲裁节点。如果无法通过常规方法（如重启故障节点）来恢复仲裁节点，则可以通过在管理器节点上运行`docker swarm init --force-new-cluster`来恢复该swarm。

```bash
# From the node to recover
docker swarm init --force-new-cluster --advertise-addr node01:2377
```

`--force-new-cluster`标志将Docker守护进程置于swarm模式。 它丢弃在丢失仲裁之前存在的成员信息，但它保留Swarm所需的数据，例如服务，任务和工作节点列表。

## 强制集群重新均衡
一般来说，你不需要强制swarm重新平衡它的任务。 当您将新节点添加到群集，或者节点在一段时间的不可用性之后重新连接到群集时，群集不会自动向空闲节点调度工作任务。如果集群为了平衡而周期性地将任务转移到不同的节点，则使用这些任务的客户端会被中断服务。为了避免中断正在运行的服务，并在整个群集中达到平衡。当新任务启动时，或者当运行任务的节点不可用时，这些任务被重新调度给不太繁忙的节点。目标是最大限度的平衡，和最小范围的服务干扰。

如果你不介意中断正在运行的任务，更关心负载平衡的状态，可以临时将服务扩展来强制进行群集负载平衡。

使用`docker service inspect --pretty <servicename>`可查看已配置的服务规模。 使用`docker service scale`时，具有最低任务数的节点将被定向为接收新的工作任务。您的群组中可能有多个负载不足的节点。您可能需要通过多次增量来扩展服务，以实现所有节点上达到平衡。

当负载平衡让您满意时，您可以将服务缩小到原始比例。您可以使用`docker service ps`来评估多节点的服务均衡情况。

参考
[`docker service scale`](../reference/commandline/service_scale.md) 和
[`docker service ps`](../reference/commandline/service_ps.md).
