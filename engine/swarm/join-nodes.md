---
description: Add worker and manager nodes to a swarm
keywords: guide, swarm mode, node
title: Join nodes to a swarm
---

当你第一次创建swarm时，把一个Docker引擎加入swarm模式。为了充分利用Swarm模式，你可以添加节点到集群：

* 添加工作节点以增加容量。将服务部署到群集时，Engine会往可用节点上调度任务，无论它们是工作节点还是管理器节点。当您将工作节点添加到集群时，可以增加集群的处理任务能力，而不会影响管理节点的raft同步。

* 管理节点可以增加容错。管理节点执行集群的编排和管理功能。在管理节点中，单个领导节点执行编排任务。如果领导节点故障，则剩余的管理节点选择新的领导者，并恢复集群状态的编排和维护。默认情况下，管理节点也会运行任务。

在将节点添加到群集之前，必须在主机上安装Docker引擎1.12或更高版本。

Docker引擎根据您提供给`docker swarm join`命令的 **join-token** 来加入swarm。节点仅在连接时使用令牌。如果随后更新令牌，不会影响现有的集群节点。请参阅[在Swarm模式下运行Docker引擎](swarm-mode.md＃view-the-join-command-or-update-a-swarm-join-token)。

## 添加工作节点

要获取包含工作节点连接令牌的join命令，请在管理节点上运行以下命令：

```bash
$ docker swarm join-token worker

To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-49nj1cmql0jkz5s954yi3oex3nedyz0fb0xx14ie39trti4wxv-8vxv8rssmk743ojnwacrr2e7c \
    192.168.99.100:2377
```

在工作节点上运行下面命令，即加入swarm集群：

```bash
$ docker swarm join \
  --token SWMTKN-1-49nj1cmql0jkz5s954yi3oex3nedyz0fb0xx14ie39trti4wxv-8vxv8rssmk743ojnwacrr2e7c \
  192.168.99.100:2377

This node joined a swarm as a worker.
```

命令 `docker swarm join` 做如下事情:

* 将当前节点上的Docker引擎切换到swarm模式。
* 从管理节点获取TLS证书。
* 命名具有机器主机名的节点。
* 基于swarm令牌，通过发布地址将当前节点加入swarm集群。
* 将当前节点设置为`活动`状态，意味着它可以从调度器接收任务。
* 将`接入`overlay网络扩展到当前节点。

### 添加管理节点
当你运行`docker swarm join`并传递管理节点令牌时，Docker引擎会切换到swarm模式管理节点。管理节点也执行raft同步。新节点是`可达`，但现有的管理节点将保持群集`领导`。

Docker建议每个集群使用三个或五个管理节点来实现高可用性。因为swarm模式管理节点使用Raft共享数据，所以必须有奇数个管理器。只要超过一半的管理器节点可用，集群就可以继续工作。

有关集群管理节点和管理集群的更多详细信息，请参阅[管理和维护一组Docker引擎](admin_guide.md)。

要获取包含管理节点连接令牌的连接命令，请在管理节点上运行以下命令：

```bash
$ docker swarm join-token manager

To add a manager to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-61ztec5kyafptydic6jfc1i33t37flcl4nuipzcusor96k7kby-5vy9t8u35tuqm7vh67lrz9xp6 \
    192.168.99.100:2377
```

执行以下命令以加入swarm集群，并成为管理节点：

```bash
$ docker swarm join \
  --token SWMTKN-1-61ztec5kyafptydic6jfc1i33t37flcl4nuipzcusor96k7kby-5vy9t8u35tuqm7vh67lrz9xp6 \
  192.168.99.100:2377

This node joined a swarm as a manager.
```

## 更多

* [命令行参考](../reference/commandline/swarm_join.md)`swarm join`
* [Swarm 模式教程](swarm-tutorial/index.md)
