---
description: expose, port, docker, bind publish
keywords: Examples, Usage, network, docker, documentation, user guide, multihost, cluster
title: Bind container ports to the host
---

本节中的信息解释Docker默认网桥中的绑定容器端口。这是一个名为`bridge`的桥接网络，在安装Docker时自动创建。

> **注意**：除了默认网桥网络之外，[Docker网络功能](../index.md)还允许您创建用户自定义的网络。

默认情况下Docker容器可以连接到外界，但是外部世界不能连接到容器。 得益于Docker服务启动时创建的`iptables`伪装规则，每个输出连接似乎源自主机的自己的IP地址之一。

```
$ sudo iptables -t nat -L -n

...
Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination
MASQUERADE  all  --  172.17.0.0/16       0.0.0.0/0
...
```
Docker服务创建一个允许容器连接到外部世界的IP地址的伪装规则。

如果您希望容器接受进入的连接，您需要在调用`docker run`时，提供特殊选项。 有两种方法。

第一种，您可以在执行`docker run`时提供`-P`或`--publish-all=true|false`选项，这是一个铺开式的操作。它会把镜像的`Dockerfile`中用`EXPOSE`行标识每个端口
或命令行中用`--expose <port>`标志的端口映射到主机 _ephemeral port range_ 内的某处。 然后，`docker port`命令可以用于检查创建的映射。 _ephemeral port range_ 是配置在`/proc/sys/net/ipv4/ip_local_port_range`内核参数中的，通常范围从32768到61000。

可以使用`-p SPEC`或`--publish = SPEC`选项明确指定映射。它允许您具体说明docker服务器上的哪个端口，可以是任何您想映射到容器中的端口，不只是在 _ephemeral port range_ 端口范围。

无论哪种方式，通过检查您的NAT表，您应该能够看到Docker在您的网络栈中完成了什么。

```
# What your NAT rules might look like when Docker
# is finished setting up a -P forward:

$ iptables -t nat -L -n

...
Chain DOCKER (2 references)
target     prot opt source               destination
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:49153 to:172.17.0.2:80

# What your NAT rules might look like when Docker
# is finished setting up a -p 80:80 forward:

Chain DOCKER (2 references)
target     prot opt source               destination
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:80 to:172.17.0.2:80
```

您可以看到Docker在`0.0.0.0`上暴露了这些容器端口。通配符将匹配主机上任何可能进入该端口的IP地址。 如果您想要更严格，只允许容器服务通过主机上的特定外部接口联系，您就有两个选择。 当您调用`docker run`时，您可以使用`-p IP:host_port:container_port`或`-p IP::port`来指定一个特定的外部接口，用于特殊的绑定。

或者如果您总是希望Docker端口转发绑定到一个特定的IP地址，您可以编辑系统范围的Docker服务设置并添加选项`--ip=IP_ADDRESS`。 记住在编辑此文件后重新启动Docker服务。

> **注意**:在启用了发卡NAT（`--userland-proxy=false`）的情况下，容器端口暴露完全通过iptables规则实现，并且不会尝试绑定暴露的端口。 这意味着没有什么可以通过为容器暴露相同的端口来防止在Docker之外的以前侦听的服务。 在这种冲突的情况下，Docker创建的iptables规则将优先路由到容器。

`--userland-proxy`参数，默认情况下为true，为容器间和外部到容器的通信提供了用户级实现。 禁用时，Docker使用附加的`MASQUERADE` iptable规则和`net.ipv4.route_localnet`内核参数，允许主机通过常用的回送地址连接到本地容器暴露端口。由于性能原因，此备选方案是首选。

## 相关信息

- [Understand Docker container networks](../index.md)
- [Work with network commands](../work-with-networks.md)
- [Legacy container links](dockerlinks.md)
