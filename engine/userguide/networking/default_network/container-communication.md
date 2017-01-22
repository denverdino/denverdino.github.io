---
description: Understand container communication
keywords: docker, container, communication, network
title: Understand container communication
---

本节中的信息解释Docker默认网桥内的容器通信。这是在安装Docker时自动创建的以`bridge`命名的`bridge`网络。

**注意**：[Docker网络功能](../index.md)允许您创建除默认bridge网络之外的自定义网络。

## 与外界通信

容器能否与外界通信是由两个因素决定的。第一个因素是主机是否转发其IP包。第二个是主机的`iptables`是否允许这个特定的连接。

IP包的转发由`ip_forward`系统参数决定。如果此参数为`1`，则数据包只能在容器之间传递。通常，您只需将Docker服务保留为默认设置，即`--ip-forward=true`，当服务启动时，Docker将为您设置`ip_forward`为`1`。如果设置`--ip-forward=false`并且系统的内核已启用，则`--ip-forward=false`选项不起作用。要检查内核上的设置或手动启动它：

```
  $ sysctl net.ipv4.conf.all.forwarding

  net.ipv4.conf.all.forwarding = 0

  $ sysctl net.ipv4.conf.all.forwarding=1

  $ sysctl net.ipv4.conf.all.forwarding

  net.ipv4.conf.all.forwarding = 1
```

> **注意** 此设置不会影响那些使用主机网络栈的容器(`--network=host`)。

许多使用Docker的人希望`ip_forward`开启，至少让容器和更广阔的外界之间的通信成为可能。如果您在多网桥设置中，也可能需要进行容器间通信。

如果给守护程序设置`--iptables=false`，Docker在启动时将不会更改您的系统`iptables`规则。否则，Docker服务将会添加转发规则到`DOCKER`过滤链。

Docker不会从DOCKER过滤链中删除或修改任何已存在的规则。这允许用户预先创建进一步限制对容器的访问所需的任何规则。

默认情况下，Docker的转发规则允许所有外部源IP。要仅允许特定IP或网络访问容器，请在DOCKER过滤链的顶部插入否决规则。例如，要限制外部访问，使只有源IP 8.8.8.8可以访问容器，可以添加以下规则：

```
$ iptables -I DOCKER -i ext_if ! -s 8.8.8.8 -j DROP
```

其中 *ext_if* 是提供与主机连接的网络接口的名称。

## 容器之间的通信

在操作系统级别，两个容器之间是否可以通信是由两个因素控制的。

- 网络拓扑是否连接容器的网络接口？默认情况下，Docker会将所有容器连接到单个`docker0`网桥，为容器之间传输数据包提供一条路径。有关其他可能的拓扑，请参阅本文档的后面部分。

- 您的`iptables`允许这个特殊的连接吗？如果在守护程序启动时设置`--iptables=false`，Docker将永远不会更改您的系统`iptables`规则。否则，如果您保留默认的`--icc=true`，则Docker服务将向FORWARD链添加ACCEPT策略的默认规则，否则如果`--icc=false`，则将策略设置为DROP。

是否保留`--icc=true`或将其更改为`--icc=false`是一个关键的问题，以便`iptables`保护其他容器和主机，避免被进行任意端口探测或攻击。

如果您选择最安全的设置，即`--icc=false`，那么容器如何在 _希望_ 它们各自提供服务的情况下进行通信呢？答案是`--link=CONTAINER_NAME_or_ID:ALIAS`选项，在上一节中提到，因为它对名称服务有影响。如果Docker守护进程运行时同时设置`--icc=false`和`--iptables=true`，那么当它看到使用`--link =`选项调用`docker run`时，Docker服务将插入一对`iptables` `ACCEPT`规则，容器可以连接到由其他容器暴露的端口——它在其`Dockerfile`的`EXPOSE`行中提到的端口。

> **注意**：`--link=`中的`CONTAINER_NAME`值必须是自动分配的Docker名称，如`stupefied_pa​​re`，或者是在运行`docker run`时指定的名称`--name=`。它不能是主机名，Docker在`--link =`选项的上下文中不能识别主机名。

您可以在Docker主机上运行`iptables`命令，以查看`FORWARD`链是否具有默认的`ACCEPT`或`DROP`策略：

```
# When --icc=false, you should see a DROP rule:

$ sudo iptables -L -n

...
Chain FORWARD (policy ACCEPT)
target     prot opt source               destination
DOCKER     all  --  0.0.0.0/0            0.0.0.0/0
DROP       all  --  0.0.0.0/0            0.0.0.0/0
...

# When a --link= has been created under --icc=false,
# you should see port-specific ACCEPT rules overriding
# the subsequent DROP policy for all other packets:

$ sudo iptables -L -n

...
Chain FORWARD (policy ACCEPT)
target     prot opt source               destination
DOCKER     all  --  0.0.0.0/0            0.0.0.0/0
DROP       all  --  0.0.0.0/0            0.0.0.0/0

Chain DOCKER (1 references)
target     prot opt source               destination
ACCEPT     tcp  --  172.17.0.2           172.17.0.3           tcp spt:80
ACCEPT     tcp  --  172.17.0.3           172.17.0.2           tcp dpt:80
```

> **注意**：Docker很小心，它的主机范围的`iptables`规则完全暴露容器间的原始IP地址，所以从一个容器到另一个容器的连接应该始终从第一个容器自己的IP地址开始。
