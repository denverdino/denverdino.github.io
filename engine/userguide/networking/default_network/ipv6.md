---
description: How do we connect docker containers within and across hosts ?
keywords: docker, network, IPv6
title: IPv6 with Docker
---

本节中的信息说明了使用Docker默认网桥的IPv6。这是在安装Docker时自动创建的以`bridge`命名的`bridge`网络。

由于我们[用尽了IPv4地址](http://en.wikipedia.org/wiki/IPv4_address_exhaustion)，IETF已经在[RFC 2460](http://en.wikipedia.org/wiki/IPv6)中标准化了IPv4后续版本的[Internet协议版本6](http://en.wikipedia.org/wiki/IPv6).这两种协议位于[OSI模型](http://en.wikipedia.org/wiki/OSI_model)的第3层。

## How IPv6 works on Docker

默认情况下，Docker服务器仅为IPv4配置容器网络。 您可以通过使用`--ipv6`标志运行Docker守护程序来启用IPv4/IPv6双栈支持。 Docker将使用IPv6[链接本地地址](http://en.wikipedia.org/wiki/Link-local_address)`fe80::1`设置桥接器`docker0`。

默认情况下，创建的容器只能获取链接本地IPv6地址。要为您的容器分配全局可路由的IPv6地址，您必须指定一个IPv6子网来选择地址。在启动Docker守护程序时通过`--fixed-cidr-v6`参数设置IPv6子网：

```
dockerd --ipv6 --fixed-cidr-v6="2001:db8:1::/64"
```

Docker容器的子网至少应该有`/80`的大小。这样，IPv6地址可以以容器的MAC地址结尾，并防止Docker层中的NDP邻居缓存无效问题。

使用`--fixed-cidr-v6`参数集，Docker将向路由表添加一个新路由。 将启用更多的IPv6路由（您可以通过使用`--ip-forward=false`启动dockerd来阻止此操作）：

```
$ ip -6 route add 2001:db8:1::/64 dev docker0

$ sysctl net.ipv6.conf.default.forwarding=1

$ sysctl net.ipv6.conf.all.forwarding=1
```

到子网`2001:db8:1::/64`的所有流量现在将通过`docker0`接口路由。

请注意，IPv6转发可能会干扰您现有的IPv6配置：如果您使用路由器广告获取主机接口的IPv6设置，您应该将accept_ra设置为2.否则启用IPv6的转发将导致拒绝路由器广播。 例如，如果您想通过路由器广告配置eth0，您应该设置：

```
$ sysctl net.ipv6.conf.eth0.accept_ra=2
```

![](images/ipv6_basic_host_config.svg)

每个新容器将从定义的子网获取IPv6地址。 此外，默认路由将通过daemon选项`--default-gateway-v6`（如果存在）指定的地址添加到容器中的`eth0`上，否则通过`fe80::1`：

```
docker run -it ubuntu bash -c "ip -6 addr show dev eth0; ip -6 route show"

15: eth0: <BROADCAST,UP,LOWER_UP> mtu 1500
   inet6 2001:db8:1:0:0:242:ac11:3/64 scope global
      valid_lft forever preferred_lft forever
   inet6 fe80::42:acff:fe11:3/64 scope link
      valid_lft forever preferred_lft forever

2001:db8:1::/64 dev eth0  proto kernel  metric 256
fe80::/64 dev eth0  proto kernel  metric 256
default via fe80::1 dev eth0  metric 1024
```

在这个例子中，Docker容器被分配一个具有网络后缀`/64`（这里：fe80::42:acff:fe11:3/64）的链接本地地址和全局可路由的IPv6地址（这里：2001:db8:0:0:242:ac11:3/64）。容器将通过`eth0`上的`fe80::1`处的链接本地网关创建到`2001:db8:1::/64`网络外部的地址的连接。

通常，服务器或虚拟机会分配一个 `/64` IPv6子网（例如`2001:db8:23:42::/64`）。 在这种情况下，您可以进一步拆分它，并提供Docker 一个 `/80` 子网，同时使用单独的`/80`子网用于主机上的其他应用程序：

![](images/ipv6_slash64_subnet_config.svg)

在此设置中，`2001:db8:23:42:0:0:0:0` 至 `2001:db8:23:42:0:ffff:ffff:ffff` 附加到eth0，主机监听 `2001:db8:23:42::1` 。`2001:db8:23:42:1:0:0:0`到`2001:db8:23:42:1:ffff:ffff`地址范围的子网`2001:db8:23:42:1::/ffff`附加到`docker0`并将被容器使用。

### 使用NDP代理

如果您的Docker主机是IPv6子网的唯一部分，但没有分配IPv6子网，则可以使用NDP代理将容器通过IPv6连接到互联网。 例如，具有IPv6地址`2001:db8::c001`的主机是子网`2001:db8::/64`的一部分，您的IaaS提供程序允许您配置IPv6地址为`2001:db8::c000`至`2001:db8::c00f`：
```
$ ip -6 addr show

1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qlen 1000
    inet6 2001:db8::c001/64 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::601:3fff:fea1:9c01/64 scope link
       valid_lft forever preferred_lft forever
```

让我们将可配置地址范围分成两个子网`2001:db8::c000/125`和`2001:db8::c008/125`。第一个可以由主机本身使用，后者由Docker使用：

```
dockerd --ipv6 --fixed-cidr-v6 2001:db8::c008/125
```

您注意到Docker子网位于由连接到`eth0`的路由器管理的子网内。这意味着所有具有来自Docker子网的地址的设备（容器）都可以在路由器子网内找到。因此路由器认为它可以直接与这些容器交谈。

![](images/ipv6_ndp_proxying.svg)

一旦路由器想要发送一个IPv6数据包到第一个容器，它会发送一个相邻请求，询问，谁有`2001:db8::c009`？ 但它将得不到回答，因为没有人在这个子网有这个地址。 具有此地址的容器隐藏在Docker主机后面。Docker主机必须监听容器地址的相邻请求，并发送本身是负责地址的设备的响应。 这通过称为`NDP代理`的内核功能来完成。 您可以通过执行下面命令来启用它

```
$ sysctl net.ipv6.conf.eth0.proxy_ndp=1
```

现在，您可以将容器的IPv6地址添加到NDP代理表：

```
$ ip -6 neigh add proxy 2001:db8::c009 dev eth0
```

此命令告诉内核回应关于设备eth0上的IPv6地址 `2001:db8::c009` 的传入邻居请求。 因此，到这个IPv6地址的所有流量将进入Docker主机，并且它将根据其路由表通过 `docker0` 设备将其转发到容器网络：
```
$ ip -6 route show

2001:db8::c008/125 dev docker0  metric 1
2001:db8::/64 dev eth0  proto kernel  metric 256
```

您必须对您的Docker子网中的每个IPv6地址执行 `ip -6 neigh add proxy ...` 命令。不幸的是，没有通过执行一个命令来添加整个子网的功能。 另一种方法是使用NDP代理守护进程，如[ndppd](https://github.com/DanielAdolfsson/ndppd)。

## Docker IPv6 集群

### 交换机网络环境
使用可路由的IPv6地址允许您实现不同主机上的容器之间的通信。 让我们来看一个简单的Docker IPv6集群示例：

![](images/ipv6_switched_network_example.svg)

Docker主机位于 `2001:db8:0::/64` 子网中。 Host1配置为提供 `2001:db8:1::/64` 子网到其容器的地址。 它有三个路由配置：

- 将所有流量通过`eth0`路由到`2001:db8:0::/64`
- 将所有流量通过`docker0`路由到`2001:db8:1::/64`
- 将所有流量通过IP为 `2001:db8::2`的Host2路由到 `2001:db8:2::/64`

Host1还充当OSI第3层上的路由器。当其中一个网络客户端尝试联系在Host1的路由表中指定的目标时，Host1将相应地转发流量。 它作为它知道的所有网络的路由器：`2001:db8::/64`， `2001:db8:1::/64` and `2001:db8:2::/64`。

在Host2上我们有几乎相同的配置。 Host2的容器将从 `2001:db8:2::/64` 获取IPv6地址。 Host2有三个路由配置：

- 将所有流量通过`eth0`路由到`2001:db8:0::/64`
- 将所有流量通过`docker0`路由到`2001:db8:2::/64`
- 将所有流量通过IP为 `2001:db8:0::1`的Host2路由到 `2001:db8:1::/64`

与Host1的区别在于网络 `2001:db8:2::/64` 通过其`docker0`接口直接连接到主机，而通过Host1的IPv6地址 `2001:db8::1` 到达 `2001:db8:1::/64` 。

这样每个容器都能够与每个其他容器接触。容器`Container1-*`共享同一子网并直接相互联系。`Container1-*` 和 `Container2-*` 之间的流量将通过Host1和Host2进行路由，因为这些容器不共享同一个子网。

在交换环境中，每个主机必须知道到每个子网的所有路由。 在向集群添加或删除主机后，您必须更新主机的路由表。

图中虚线下方的每个配置由Docker处理：`docker0` 网桥IP地址配置，到主机上的Docker子网的路由，容器IP地址和容器上的路由。虚线上方的配置取决于用户，并且可以适应个别环境。

### 路由器网络环境
在路由网络环境中，您将第2层交换机替换为第3层路由器。 现在主机只需要知道他们的默认网关（路由器）和到自己的容器（由Docker管理）的路由。 路由器保存有关Docker子网的所有路由信息。 当您向此环境添加或删除主机时，您只需更新路由器中的路由表——而不是每个主机上的路由表。

![](images/ipv6_routed_network_example.svg)

在这种情况下，同一主机的容器可以直接相互通信。不同主机上的容器之间的流量将通过其主机和路由器进行路由。例如，从`Container1-1`到`Container2-1`的分组将通过`Host1`，`Router`和`Host2`路由，直到它到达`Container2-1`。

为了保持IPv6地址简短，在这个示例中，将`a/48`网络分配给每个主机。主机使用`/64`子网给自己的服务，一个给Docker。当添加第三台主机时，您将在路由器中为子网 `2001:db8:3::/48` 添加一个路由，并使用 `--fixed-cidr-v6=2001:db8:3:1::/64` 配置Host3上的Docker 。

记住Docker容器的子网至少应该有 `/80` 的大小。这样，IPv6地址可以以容器的MAC地址结尾，并防止Docker层中的NDP邻居缓存无效问题。所以如果您有一个 `/64` ，为您的整个环境使用 `/78` 子网的主机和 `/80` 的容器。这样，您可以使用4096个主机，每个主机具有16个 `/80` 个子网。

图中虚线下方的每个配置由Docker处理：`docker0` bridge IP地址配置，到主机上的Docker子网的路由，容器IP地址和容器上的路由。虚线上方的配置取决于用户，并且可以适应个别环境。
