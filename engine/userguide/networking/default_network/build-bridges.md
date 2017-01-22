---
description: Learn how to build your own bridge interface
keywords: docker, bridge, docker0, network
title: Build your own bridge
---

本节介绍如何构建自己的网桥来替换Docker默认网桥。这是在安装Docker时自动创建的以`bridge`命名的网桥。

> **注意**：[Docker网络功能](../index.md)允许您创建除默认网桥网络之外的用户自定义网络。

可以在启动Docker之前设置您自己的网桥，并使用`-b BRIDGE`或`--bridge=BRIDGE`告诉Docker使用您自定义的网桥。如果已经有Docker以默认配置`docker0`启动并运行，您可以直接创建您的网桥并重新启动Docker，或者通过停止服务和删除接口开始：

```
# Stopping Docker and removing docker0

$ sudo service docker stop

$ sudo ip link set dev docker0 down

$ sudo brctl delbr docker0

$ sudo iptables -t nat -F POSTROUTING
```

然后，在启动Docker服务之前，创建自己的网桥，并给它任何您想要的配置。 这里我们，可以使用上一节中的选项来自定义`docker0`，从而创建一个足够简单的网桥，它足以说明该技术。

```
# Create our own bridge

$ sudo brctl addbr bridge0

$ sudo ip addr add 192.168.5.1/24 dev bridge0

$ sudo ip link set dev bridge0 up

# Confirming that our bridge is up and running

$ ip addr show bridge0

4: bridge0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state UP group default
    link/ether 66:38:d0:0d:76:18 brd ff:ff:ff:ff:ff:ff
    inet 192.168.5.1/24 scope global bridge0
       valid_lft forever preferred_lft forever

# Tell Docker about it and restart (on Ubuntu)

$ echo 'DOCKER_OPTS="-b=bridge0"' >> /etc/default/docker

$ sudo service docker start

# Confirming new outgoing NAT masquerade is set up

$ sudo iptables -t nat -L -n

...
Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination
MASQUERADE  all  --  192.168.5.0/24      0.0.0.0/0
```

结果应该是Docker服务成功启动，准备将容器绑定到新的网桥。在暂停以验证网桥的配置后，尝试创建一个容器——您将看到它的IP地址在您的新IP地址范围内，Docker将自动检测该IP地址范围。

您可以使用`brctl show`命令查看Docker在启动和停止容器时添加和删除接口，并且可以在容器中运行`ip addr`和`ip route`以查看它是否已在网桥的IP地址范围中给出了地址，并已被告知使用Docker主机在网桥上的IP地址作为其余Internet的默认网关。
