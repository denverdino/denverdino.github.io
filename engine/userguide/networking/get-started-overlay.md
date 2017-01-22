---
description: Use overlay for multi-host networking
keywords: Examples, Usage, network, docker, documentation, user guide, multihost, cluster
title: Get started with multi-host networking
---

本文将通过示例来为你说明创建多主机网络的一些基础知识。
Docker Engine通过`overlay`网络驱动程序支持多主机网络开箱即用。
与`bridge`网络驱动不同，在创建`overlay`网络之前，需要一些先决条件：

* [Docker Engine running in swarm mode](#overlay-networking-and-swarm-mode)

或

* [A cluster of hosts using a key value store](#overlay-networking-with-an-external-key-value-store)

## Overlay网络和swarm mode

使用[swarm mode](../../ swarm/swarm-mode.md)管理下运行的docker engine，可以在管理器节点上创建`overlay`v网络。

swarm下创建的`overlay`网络仅对swarm管理下的节点可用。
当你创建一个使用`overlay`网络的服务时，管理节点将会自动将你运行该服务的节点加入到`overlay`网络中。

要了解有关在swarm mode下运行Docker Engine的更多信息，请参阅[Swarm mode overview](../../swarm/index.md)。

下面的示例将展示如何创建`overlay`网络，并将其用于在swarm中管理的节点上的服务：

```bash
# Create an overlay network `my-multi-host-network`.
$ docker network create \
  --driver overlay \
  --subnet 10.0.9.0/24 \
  my-multi-host-network

400g6bwzd68jizzdx5pgyoe95

# Create an nginx service and extend the my-multi-host-network to nodes where
# the service's tasks run.
$ docker service create --replicas 2 --network my-multi-host-network --name my-web nginx

716thylsndqma81j6kkkb5aus
```

对于一个集群的`overlay`网络，不可用将其用于一个非集群管理的容器。
有关更多信息，请参阅[Docker swarm mode overlay network security model](overlay-security-model.md)。

另请参见[Attach services to an overlay network](../../swarm/networking.md)。

## 依赖外置的key-value存储的`Overlay`网络

要使用依赖外部key-value存储的Docker Engine，您需要以下内容：

* 可访问的key-value存储。 Docker支持Consul，Etcd和ZooKeeper（分布式存储）key-value存储。
* 在集群的每个主机可以连接到key-value存储。
* 在集群的每个主机都正确配置了Docker daemon。
* 群集中的主机必须具有唯一的主机名，因为key-value存储使用主机名来标识集群成员。

尽管试用key-value存储的Docker多主机网络时，Docker Machine和Docker Swarm不是必须的。
但在本示例中，我们将使用它们来说明它们是如何集成的。 您将使用Machine创建键值存储服务器和主机集群。 此示例创建一个群集群。

>**注意：** 在swarm mode下运行的Docker Engine与使用外部key-value存储管理的网络不兼容。

### 前置依赖

在开始之前，请确保您的网络上有一个安装了最新版本的Docker Engine和Docker Machine的系统，该示例还依赖于VirtualBox。
如果您在Mac或Windows上使用Docker Toolbox的话，那您就已经安装了所有需要的东西。

如果您还没有这样做，请确保将Docker Engine和Docker Machine升级到最新版本。

### 设置key-value存储

`overlay`网络依赖key-value存储。key-value存储保存了有关网络状态的一些信息，包括发现，网络，端点，IP地址等。
Docker支持Consul，Etcd和ZooKeeper键值存储。 这个例子使用了Consul。

1. 登录到已经做好前置依赖准备的系统上，安装好Docker Engine，Docker Machine，VirtualBox的系统

2. 提供一个名为`mh-keystore`的VirtualBox虚拟机。

		$ docker-machine create -d virtualbox mh-keystore

    当你准备一个新的机器时，该进程会将Docker Engine添加到主机。
    这意味着，不是手动安装Consul，您可以使用[来自Docker Hub的consul镜像](https://hub.docker.com/r/progrium/consul/)来创建一个实例。
    您将在下一步中执行此操作。

3. 设置你的本地环境为`mh-keystore`机器的环境

		$  eval "$(docker-machine env mh-keystore)"

4. 在`mh-keystore`机器上启动一个`progrium/consul`的容器

		$  docker run -d \
			-p "8500:8500" \
			-h "consul" \
			progrium/consul -server -bootstrap

    客户端启动在`mh-keystore`机器运行的`progrium/consul`镜像。
    服务器被称为`consul`，正在监听`8500`端口。

5. 允许`docker ps`命令可以看到正在运行的`consul`容器

		$ docker ps

		CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                                                                            NAMES
		4d51392253b3        progrium/consul     "/bin/start -server -"   25 minutes ago      Up 25 minutes       53/tcp, 53/udp, 8300-8302/tcp, 0.0.0.0:8500->8500/tcp, 8400/tcp, 8301-8302/udp   admiring_panini

继续保持你打开着的终端，进入到下一节


### 创建swarm集群

在此步骤中，您将使用`docker-machine`为您的网络配置主机。此时，您不会真正的创建网络。
您将在VirtualBox中创建几台机器。你将创建的第一个机器为主(master)。
在创建每个主机时，您将提供`overlay`网络相关的参数。

1. 创建Swarm master

		$ docker-machine create \
		-d virtualbox \
		--swarm --swarm-master \
		--swarm-discovery="consul://$(docker-machine ip mh-keystore):8500" \
		--engine-opt="cluster-store=consul://$(docker-machine ip mh-keystore):8500" \
		--engine-opt="cluster-advertise=eth1:2376" \
		mhs-demo0

    在创建机器时，您将为`daemon`提供`--cluster-store`选项。此选项将确定`overlay`网络的键值存储的位置。
    bash表达式`$（docker-machine ip mh-keystore）`将解析为在`STEP 1`中创建的Consul服务器的IP地址。
    `--cluster-advertise`选项在网络上进行广播。

2. 创建其他机器并加入Swarm集群

		$ docker-machine create -d virtualbox \
			--swarm \
			--swarm-discovery="consul://$(docker-machine ip mh-keystore):8500" \
			--engine-opt="cluster-store=consul://$(docker-machine ip mh-keystore):8500" \
			--engine-opt="cluster-advertise=eth1:2376" \
		  mhs-demo1

3. 列出你的所有机器，确认他们都在运行中

		$ docker-machine ls

		NAME         ACTIVE   DRIVER       STATE     URL                         SWARM
		default      -        virtualbox   Running   tcp://192.168.99.100:2376
		mh-keystore  *        virtualbox   Running   tcp://192.168.99.103:2376
		mhs-demo0    -        virtualbox   Running   tcp://192.168.99.104:2376   mhs-demo0 (master)
		mhs-demo1    -        virtualbox   Running   tcp://192.168.99.105:2376   mhs-demo0

此时，您有一组在网络上运行的主机。 您已准备好为使用这些主机的容器创建多主机网络的环境了。

保持您的终端打开并进入下一步。

### 创建一个overlay网络

为了创建一个overlay网络

1. 将你宿主机的环境设置为swarm master机器的环境

		$ eval $(docker-machine env --swarm mhs-demo0)

    在`docker-machine`命令中使用`--swarm`标志可以限制`docker`命令只展示swarm相关信息。

2. 使用`docker info`命令来看一下swarm集群的状态

		$ docker info

		Containers: 3
		Images: 2
		Role: primary
		Strategy: spread
		Filters: affinity, health, constraint, port, dependency
		Nodes: 2
		mhs-demo0: 192.168.99.104:2376
		└ Containers: 2
		└ Reserved CPUs: 0 / 1
		└ Reserved Memory: 0 B / 1.021 GiB
		└ Labels: executiondriver=native-0.2, kernelversion=4.1.10-boot2docker, operatingsystem=Boot2Docker 1.9.0 (TCL 6.4); master : 4187d2c - Wed Oct 14 14:00:28 UTC 2015, provider=virtualbox, storagedriver=aufs
		mhs-demo1: 192.168.99.105:2376
		└ Containers: 1
		└ Reserved CPUs: 0 / 1
		└ Reserved Memory: 0 B / 1.021 GiB
		└ Labels: executiondriver=native-0.2, kernelversion=4.1.10-boot2docker, operatingsystem=Boot2Docker 1.9.0 (TCL 6.4); master : 4187d2c - Wed Oct 14 14:00:28 UTC 2015, provider=virtualbox, storagedriver=aufs
		CPUs: 2
		Total Memory: 2.043 GiB
		Name: 30438ece0915

    从此信息，您可以看到集群中运行着三个容器，然后两个运行在Masters节点上。

3. 创建你的`overlay`网络

		$ docker network create --driver overlay --subnet=10.0.9.0/24 my-net

    您只需在群集中的某一台机器上创建网络。
    在这个例子中，您使用了swarm master机器运行它，但您实际上可以在群集中的任何主机上运行它。

> **Note** : 强烈建议你在创建网络的时候使用`--subnet`参数。如果不指定`--subnet`参数，Docker Daemon将为该网络自动选择一个子网。
> 这很可能会和你基础设施中另一个不由docker管理的子网相重叠，当容器连接到这样的网络时，这种子网的重叠很有可能导致连接问题或故障。

4. 检查网络是否正在运行

		$ docker network ls

		NETWORK ID          NAME                DRIVER
		412c2496d0eb        mhs-demo1/host      host
		dd51763e6dd2        mhs-demo0/bridge    bridge
		6b07d0be843f        my-net              overlay
		b4234109bd9b        mhs-demo0/none      null
		1aeead6dd890        mhs-demo0/host      host
		d0bb78cbe7bd        mhs-demo1/bridge    bridge
		1c0eb8f69ebb        mhs-demo1/none      null

    由于您处于swarm master的环境中，您将看到所有swarm agent上的所有网络：
    每个Docker Engine上的默认网络和单个`overlay`网络。请注意，每个`NETWORK ID`是唯一的。

5. 分别切换到每一个swarm agent，列出他们的网络

		$ eval $(docker-machine env mhs-demo0)

		$ docker network ls

		NETWORK ID          NAME                DRIVER
		6b07d0be843f        my-net              overlay
		dd51763e6dd2        bridge              bridge
		b4234109bd9b        none                null
		1aeead6dd890        host                host

		$ eval $(docker-machine env mhs-demo1)

		$ docker network ls

		NETWORK ID          NAME                DRIVER
		d0bb78cbe7bd        bridge              bridge
		1c0eb8f69ebb        none                null
		412c2496d0eb        host                host
		6b07d0be843f        my-net              overlay

    每一个agent都返回了`my-net`网络，并且`NETWORK ID`是一致的。
    现在你就有了一个正在运行中的跨主机网络了。

### 在你的网络上运行一个应用

当你的网络创建之后，您就可以在任何主机上启动容器，并将其自动加入到网络中。
Once your network is created, you can start a container on any of the hosts and it automatically is part of the network.

1. 将本地环境切换到Swarm master的环境

		$ eval $(docker-machine env --swarm mhs-demo0)

2. 在`mhs-demo0`实例上启动一个Nginx服务

		$ docker run -itd --name=web --network=my-net --env="constraint:node==mhs-demo0" nginx

3. 启动一个BusyBox的服务在`mhs-demo1`实例上，然后获取一下Nginx主页的内容

		$ docker run -it --rm --network=my-net --env="constraint:node==mhs-demo1" busybox wget -O- http://web

		Unable to find image 'busybox:latest' locally
		latest: Pulling from library/busybox
		ab2b8a86ca6c: Pull complete
		2c5ac3f849df: Pull complete
		Digest: sha256:5551dbdfc48d66734d0f01cafee0952cb6e8eeecd1e2492240bf2fd9640c2279
		Status: Downloaded newer image for busybox:latest
		Connecting to web (10.0.0.2:80)
		<!DOCTYPE html>
		<html>
		<head>
		<title>Welcome to nginx!</title>
		<style>
		body {
				width: 35em;
				margin: 0 auto;
				font-family: Tahoma, Verdana, Arial, sans-serif;
		}
		</style>
		</head>
		<body>
		<h1>Welcome to nginx!</h1>
		<p>If you see this page, the nginx web server is successfully installed and
		working. Further configuration is required.</p>

		<p>For online documentation and support please refer to
		<a href="http://nginx.org/">nginx.org</a>.<br/>
		Commercial support is available at
		<a href="http://nginx.com/">nginx.com</a>.</p>

		<p><em>Thank you for using nginx.</em></p>
		</body>
		</html>
		-                    100% |*******************************|   612   0:00:00 ETA


### 检查外部连接

正如您所见，Docker的内置`overlay`网络驱动程序在同一网络中的多个主机上的容器之间提供了开箱即用的连接。
此外，连接到多主机网络的容器也会自动连接到`docker_gwbridge`网络。
此网络允许容器链接到外部网络。

1. 切换你的环境到swarm agent节点

		$ eval $(docker-machine env mhs-demo1)

2. 列出当前网络，你会发现`docker_gwbridge`网络

		$ docker network ls

		NETWORK ID          NAME                DRIVER
		6b07d0be843f        my-net              overlay
		dd51763e6dd2        bridge              bridge
		b4234109bd9b        none                null
		1aeead6dd890        host                host
		e1dbd5dff8be        docker_gwbridge     bridge

3. 同理，重复上面两步，观察swarm master节点

		$ eval $(docker-machine env mhs-demo0)

		$ docker network ls

		NETWORK ID          NAME                DRIVER
		6b07d0be843f        my-net              overlay
		d0bb78cbe7bd        bridge              bridge
		1c0eb8f69ebb        none                null
		412c2496d0eb        host                host
		97102a22e8d2        docker_gwbridge     bridge

4. 检查Nginx容器的网络接口

		$ docker exec web ip addr

		1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default
		link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
		inet 127.0.0.1/8 scope host lo
		    valid_lft forever preferred_lft forever
		inet6 ::1/128 scope host
		    valid_lft forever preferred_lft forever
		22: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default
		link/ether 02:42:0a:00:09:03 brd ff:ff:ff:ff:ff:ff
		inet 10.0.9.3/24 scope global eth0
		    valid_lft forever preferred_lft forever
		inet6 fe80::42:aff:fe00:903/64 scope link
		    valid_lft forever preferred_lft forever
		24: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
		link/ether 02:42:ac:12:00:02 brd ff:ff:ff:ff:ff:ff
		inet 172.18.0.2/16 scope global eth1
		    valid_lft forever preferred_lft forever
		inet6 fe80::42:acff:fe12:2/64 scope link
		    valid_lft forever preferred_lft forever

    `eth0`接口表示连接了`my-net`网络。
    `eth1`接口表示连接了`docker_gwbridge`网络。

### Docker Compose相关

请参阅[Compose V2格式](https://docs.docker.com/compose/networking/)中介绍的网络功能。
并在上述多主机网络场景中执行。

## 相关文档

* [Understand Docker container networks](index.md)
* [Work with network commands](work-with-networks.md)
* [Docker Swarm overview](/swarm)
* [Docker Machine overview](/machine)
