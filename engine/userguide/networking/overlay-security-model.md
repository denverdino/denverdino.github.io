---
description: Docker swarm mode overlay network security model
keywords:
- network, docker, documentation, user guide, multihost, swarm mode
- overlay
title: Docker swarm mode overlay network security model
---

swarm mode创建的集群提供的`overlay`网络，其安全也是开箱即用的。
集群节点使用gossip协议交换`overlay`网络的数据。
默认情况下，集群节点使用GCM模式下的[AES算法](https://en.wikipedia.org/wiki/Galois/Counter_Mode)进行加密和验证。
群中的管理器节点每12小时更新一次用于gossip加密的秘钥。

您还可以加密`overlay`网络上不同节点上的容器之间传输的数据。
要启用加密，您需要在创建`overlay`网络时添加`--opt encrypted`标志：

```bash
$ docker network create --opt encrypted --driver overlay my-multi-host-network

dt0zvqn0saezzinc8a5g4worx
```

当启用`overlay`加密时，Docker会在所有节点之间创建IPSEC隧道，这个任务会被下发到每一个连接到`overlay`网络的服务。
这些隧道还在GCM模式中使用AES算法进行加密，并且管理器节点每12小时自动更新密钥。


## Swarm mode下的overlay网络和非托管容器

因为swarm模式的`overlay`网络使用来自管理器节点的加密密钥来加密gossip通信，
所以只有作为集群中的任务运行的容器才能访问密钥。
因此，使用`docker run`（非托管容器）在swarm模式之外启动的容器不能添加到`overlay`网络中。

例如：

```bash
$ docker run --network my-multi-host-network nginx

docker: Error response from daemon: swarm-scoped network
(my-multi-host-network) is not compatible with `docker create` or `docker
run`. This network can only be used by a docker service.
```

要解决此问题，请将非托管容器迁移到Swarm模式下管理。 例如：

```bash
$ docker service create --network my-multi-host-network my-image
```

因为[swarm模式](../../swarm/index.md)是一个可选功能，所以Docker Engine保留了向后兼容性。
如果愿意，您可以继续依赖第三方键值存储来支持`overlay`网络。
但是，强烈建议用户切换到swarm模式。除了本文中描述的安全优势之外，
swarm模式还使您能够利用新的服务相关的API，提供的更大的可扩展性。
