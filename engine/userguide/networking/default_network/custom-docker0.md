---
description: Customizing docker0
keywords: docker, bridge, docker0, network
title: Customize the docker0 bridge
---

本节中的信息说明如何自定义Docker默认网桥。 这是在安装Docker时自动创建的以`bridge`命名的`bridge`网络。

**注意**：[Docker网络功能](../index.md)允许您创建除默认网桥网络之外的用户定义网络。

默认情况下，Docker服务器创建和配置主机系统的`docker0`接口作为Linux内核中的 _Ethernet bridge_ 网桥，可以在其他物理或虚拟网络接口之间来回传递数据包，使其作为单个以太网网络运行。

Docker使用IP地址，网络掩码和IP分配范围来配置`docker0`。主机可以接收和发送分组包到连接到网桥的容器，并且给它一个MTU——接口允许的最大传输单元或最大分组长度——1,500字节。这些选项可在服务器启动时配置：
- `--bip=CIDR`——使用标准CIDR符号（如192.168.1.5/24）为`docker0`网桥提供特定的IP地址和网络掩码。

- `--fixed-cidr=CIDR`——使用标准CIDR符号（如172.16.1.0/28）从`docker0`子网限制IP范围。此范围必须是固定IP（例如：10.20.0.0/16）的IPv4范围，并且必须是网桥IP范围的一个子集（`docker0`或由`--bridge`设置的）。 例如，`--fixed-cidr=192.168.1.0/25`，您的容器的IP将从`192.168.1.0/24`子网的前半部分中选择。

- `--mtu=BYTES` ——覆盖`docker0`上的最大包长度。

一旦启动并运行一个或多个容器，您可以通过在主机上运行`brctl`命令并查看输出的`interfaces`列，确认Docker已正确连接到`docker0`网桥。这里是连接两个不同容器的主机：

```
# Display bridge info

$ sudo brctl show

bridge name     bridge id               STP enabled     interfaces
docker0         8000.3a1d7362b4ee       no              veth65f9
                                                        vethdda6
```

如果在Docker主机上没有安装`brctl`命令，那么在Ubuntu上您应该可以运行`sudo apt-get install bridge-utils`来安装它。

最后，每次创建新容器时都使用`docker0`以太网网桥设置。Docker在您每次执行`docker run`命令运行新容器时从网桥中选择一个可用的IP地址，并使用该IP地址和网桥的网络掩码配置容器的`eth0`接口。Docker主机在网桥上的自己的IP地址被用作每个容器到达互联网其余部分的默认网关。

```
# The network, as seen from a container

$ docker run -i -t --rm base /bin/bash

root@f38c87f2a42d:/# ip addr show eth0

24: eth0: <BROADCAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 32:6f:e0:35:57:91 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.3/16 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::306f:e0ff:fe35:5791/64 scope link
       valid_lft forever preferred_lft forever

root@f38c87f2a42d:/# ip route

default via 172.17.42.1 dev eth0
172.17.0.0/16 dev eth0  proto kernel  scope link  src 172.17.0.3

root@f38c87f2a42d:/# exit
```

请记住，Docker主机不会愿意将容器包转发到Internet，除非其`ip_forward`系统设置为1——有关详细信息，请参阅[与外部通信](container-communication.md#communicating-to-the-outside-world)一节。
