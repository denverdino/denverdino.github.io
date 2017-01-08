---
description: How to work with docker networks
keywords: commands, Usage, network, docker, cluster
title: Work with network commands
---

这篇文章提供了网络相关命令的一些示例，你可以通过它和Docker网络和容器进行交互。
以下命令可以通过Docker客户端来使用：

* `docker network create`
* `docker network connect`
* `docker network ls`
* `docker network rm`
* `docker network disconnect`
* `docker network inspect`

虽然不是必须的, 但你最好在尝试这些命令之前，先阅读一下这篇文章[Understanding Dockernetwork](index.md)。
这篇文章提供了`bridge`网络的示例，你可以立刻试验他们。
类似的，如果你跟希望试验`overlay`网络，你可以先阅读一下这篇文档[Getting started with multi-host networks](get-started-overlay.md)

## 创建网络

在你安装Docker Engine的时候，一个默认的`bridge`网络就会被自动创建了。
这个网络对应了Docker Engine传统的`docker0`网桥。
除此之外，你还可以创建自己的`bridge`或`overlay`网络。

一个`bridge`网络只会存在于一个运行着Docker Engine的宿主机上。
而一个`overlay`网络可以横跨在多个运行着Docker Engine的宿主机之间。
当你运行`docker network create`命令时，如果只简单的提供一个网络名称的参数，那么这条命令将为你创建一个`bridge`网络。

```bash
$ docker network create simple-network

69568e6336d8c96bbf57869030919f7c69524f71183b44d80948bd3927c87f6a

$ docker network inspect simple-network
[
    {
        "Name": "simple-network",
        "Id": "69568e6336d8c96bbf57869030919f7c69524f71183b44d80948bd3927c87f6a",
        "Scope": "local",
        "Driver": "bridge",
        "IPAM": {
            "Driver": "default",
            "Config": [
                {
                    "Subnet": "172.22.0.0/16",
                    "Gateway": "172.22.0.1"
                }
            ]
        },
        "Containers": {},
        "Options": {},
        "Labels": {}
    }
]
```

和`bridge`网络不同，`overlay`网络的创建则需要一些前置依赖。
在你想要创建一个`overlay`网络之前，你需要满足以下条件：

* 一个可访问的key-value存储。Docker Engine支持Consul、Etcd、ZooKeeper这几种key-value存储。
* 同一个集群的多台机器都能连接到这个key-value存储。
* 集群中的每一台机器的`daemon`都进行正确的配置。

`dockerd`关于`overlay`的配置参数有以下几个：

* `--cluster-store`
* `--cluster-store-opt`
* `--cluster-advertise`

虽然不是必需的，但是我们推荐你安装Docker Swarm来管理集群。
Swarm提供了更成熟的服务发现和服务器管理能力，可以帮助您的实现这些。

创建网络时，Docker Engine默认会创建一个不重叠的子网。当然你可以通过`--subnet`参数指定自己需要的子网。
在一个`bridge`网络上只能创建一个子网。在`overlay`网络上可以创建多个子网。

> **Note** : 强烈建议你在创建网络的时候使用`--subnet`参数。如果不指定`--subnet`参数，Docker Daemon将为该网络自动选择一个子网。
> 这很可能会和你基础设施中另一个不由docker管理的子网相重叠，当容器连接到这样的网络时，这种子网的重叠很有可能导致连接问题或故障。

除了`--subnet`参数, 你还有可能使用到这几个参数 `--gateway`，`--ip-range`，`--aux-address`。

```bash
$ docker network create -d overlay \
  --subnet=192.168.0.0/16 \
  --subnet=192.170.0.0/16 \
  --gateway=192.168.0.100 \
  --gateway=192.170.0.100 \
  --ip-range=192.168.1.0/24 \
  --aux-address="my-router=192.168.1.5" --aux-address="my-switch=192.168.1.6" \
  --aux-address="my-printer=192.170.1.5" --aux-address="my-nas=192.170.1.6" \
  my-multihost-network
```

创建网络时，请确保你配置的子网不会重叠。如果重叠，创建将会失败并返回错误信息。

When creating a custom network, the default network driver (i.e. `bridge`) has
additional options that can be passed. The following are those options and the
equivalent docker daemon flags used for docker0 bridge:

| Option                                           | Equivalent  | Description                                           |
|--------------------------------------------------|-------------|-------------------------------------------------------|
| `com.docker.network.bridge.name`                 | -           | bridge name to be used when creating the Linux bridge |
| `com.docker.network.bridge.enable_ip_masquerade` | `--ip-masq` | Enable IP masquerading                                |
| `com.docker.network.bridge.enable_icc`           | `--icc`     | Enable or Disable Inter Container Connectivity        |
| `com.docker.network.bridge.host_binding_ipv4`    | `--ip`      | Default IP when binding container ports               |
| `com.docker.network.driver.mtu`                  | `--mtu`     | Set the containers network MTU                        |

`com.docker.network.driver.mtu`参数只适用于`overlay`网络。

以下参数可以在使用`docker network create`时使用，适用于各种网络。

| Argument     | Equivalent | Description                              |
|--------------|------------|------------------------------------------|
| `--internal` | -          | Restricts external access to the network |
| `--ipv6`     | `--ipv6`   | Enable IPv6 networking                   |

举个栗子, 现在我们使用`-o`或者`--opt`参数，来指定容器绑定端口时使用一个特定的IP

```bash
$ docker network create -o "com.docker.network.bridge.host_binding_ipv4"="172.23.0.1" my-network

b1a086897963e6a2e7fc6868962e55e746bee8ad0c97b54a5831054b5f62672a

$ docker network inspect my-network

[
    {
        "Name": "my-network",
        "Id": "b1a086897963e6a2e7fc6868962e55e746bee8ad0c97b54a5831054b5f62672a",
        "Scope": "local",
        "Driver": "bridge",
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.23.0.0/16",
                    "Gateway": "172.23.0.1"
                }
            ]
        },
        "Containers": {},
        "Options": {
            "com.docker.network.bridge.host_binding_ipv4": "172.23.0.1"
        },
        "Labels": {}
    }
]

$ docker run -d -P --name redis --network my-network redis

bafb0c808c53104b2c90346f284bda33a69beadcab4fc83ab8f2c5a4410cd129

$ docker ps

CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                        NAMES
bafb0c808c53        redis               "/entrypoint.sh redis"   4 seconds ago       Up 3 seconds        172.23.0.1:32770->6379/tcp   redis
```

## 连接容器

您可以将容器动态连接到一个或多个网络。这些网络可以支持相同或不同的网络驱动程序。
一旦连接到网络，容器可以使用另一个容器的IP地址或名称进行通信。

对于`overlay`网络或是可以跨主机网络访问的用户自定义的网络插件。
容器连接到这样的跨主机网络中之后，也可以以同样的方式进行通信。

创建两个容器的例子:

```bash
$ docker run -itd --name=container1 busybox

18c062ef45ac0c026ee48a83afa39d25635ee5f02b58de4abc8f467bcaa28731

$ docker run -itd --name=container2 busybox

498eaaaf328e1018042c04b2de04036fc04719a6e39a097a4f4866043a2c2152
```

然后，我们再创建一个隔离的`bridge`再测试。


```bash
$ docker network create -d bridge --subnet 172.25.0.0/16 isolated_nw

06a62f1c73c4e3107c0f555b7a5f163309827bfbbf999840166065a8f35455a8
```

连接`container2`到这个网络中，使用`inspect`命令确认连接状态。

```
$ docker network connect isolated_nw container2

$ docker network inspect isolated_nw

[
    {
        "Name": "isolated_nw",
        "Id": "06a62f1c73c4e3107c0f555b7a5f163309827bfbbf999840166065a8f35455a8",
        "Scope": "local",
        "Driver": "bridge",
        "IPAM": {
            "Driver": "default",
            "Config": [
                {
                    "Subnet": "172.25.0.0/16",
                    "Gateway": "172.25.0.1"
                }
            ]
        },
        "Containers": {
            "90e1f3ec71caf82ae776a827e0712a68a110a3f175954e5bd4222fd142ac9428": {
                "Name": "container2",
                "EndpointID": "11cedac1810e864d6b1589d92da12af66203879ab89f4ccd8c8fdaa9b1c48b1d",
                "MacAddress": "02:42:ac:19:00:02",
                "IPv4Address": "172.25.0.2/16",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {}
    }
]
```

您可以看到Docker Engine自动为`container2`分配了一个IP地址。
根据在创建网络时指定的`--subnet`，Docker Engine从这个子网中选择了一个地址。
现在，我们再启动第三个容器，并使用`docker run`命令的`--network`选项将其连接到网络上：

```bash
$ docker run --network=isolated_nw --ip=172.25.3.3 -itd --name=container3 busybox

467a7863c3f0277ef8e661b38427737f28099b61fa55622d6c30fb288d88c551
```

如上你能够指定你的容器的IP地址。
只要容器连接的网络是用户创建并指定了子网的，您就可以在执行`docker run`和`docker network connect`时为容器选择IPv4和/或IPv6地址。
命令分别通过IPv4和IPv6的`-ip`和`-ip6`参数。所选的IP地址必须在容器网络指定的子网中，并且将在容器重新加载时保留。
该功能仅在用户定义网络中可用，因为它们保证子网配置不会在守护程序重新加载时被更改。

现在，通过`inspect`命令查看`container3`容器使用的网络。

```bash
$ docker inspect --format='{{json .NetworkSettings.Networks}}'  container3

{"isolated_nw":{"IPAMConfig":{"IPv4Address":"172.25.3.3"},"NetworkID":"1196a4c5af43a21ae38ef34515b6af19236a3fc48122cf585e3f3054d509679b",
"EndpointID":"dffc7ec2915af58cc827d995e6ebdc897342be0420123277103c40ae35579103","Gateway":"172.25.0.1","IPAddress":"172.25.3.3","IPPrefixLen":16,"IPv6Gateway":"","GlobalIPv6Address":"","GlobalIPv6PrefixLen":0,"MacAddress":"02:42:ac:19:03:03"}}
```

再来一次，我们再使用这个命令看一下`container2`容器，如果你的系统安装了Python，那么你可以格式化一下输出内容。

```bash
$ docker inspect --format='{{json .NetworkSettings.Networks}}'  container2 | python -m json.tool

{
    "bridge": {
        "NetworkID":"7ea29fc1412292a2d7bba362f9253545fecdfa8ce9a6e37dd10ba8bee7129812",
        "EndpointID": "0099f9efb5a3727f6a554f176b1e96fca34cae773da68b3b6a26d046c12cb365",
        "Gateway": "172.17.0.1",
        "GlobalIPv6Address": "",
        "GlobalIPv6PrefixLen": 0,
        "IPAMConfig": null,
        "IPAddress": "172.17.0.3",
        "IPPrefixLen": 16,
        "IPv6Gateway": "",
        "MacAddress": "02:42:ac:11:00:03"
    },
    "isolated_nw": {
        "NetworkID":"1196a4c5af43a21ae38ef34515b6af19236a3fc48122cf585e3f3054d509679b",
        "EndpointID": "11cedac1810e864d6b1589d92da12af66203879ab89f4ccd8c8fdaa9b1c48b1d",
        "Gateway": "172.25.0.1",
        "GlobalIPv6Address": "",
        "GlobalIPv6PrefixLen": 0,
        "IPAMConfig": null,
        "IPAddress": "172.25.0.2",
        "IPPrefixLen": 16,
        "IPv6Gateway": "",
        "MacAddress": "02:42:ac:19:00:02"
    }
}
```

你会发现`container2`属于两个网络。
一个是`bridge`网络，当你启动容器时，容器默认加入的网络。
另一个是`isolated_nw`网络，你后续用命令连接上的。

![](images/working.png)

再看一下`container3`容器，因为你使用`docker run`启动容器的时候将容器连接到了`isolated_nw`网络，所以它没有连接到默认的`bridge`网络。

使用`docker attach`命令，登陆到`container2`容器，然后检查一下网络栈：

```bash
$ docker attach container2
```

如果您查看容器的网络栈，您应该会看到两个以太网接口，一个是默认`bridge`网络，一个连接`isolated_nw`网络。

```bash
/ # ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:AC:11:00:03
          inet addr:172.17.0.3  Bcast:0.0.0.0  Mask:255.255.0.0
          inet6 addr: fe80::42:acff:fe11:3/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:9001  Metric:1
          RX packets:8 errors:0 dropped:0 overruns:0 frame:0
          TX packets:8 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:648 (648.0 B)  TX bytes:648 (648.0 B)

eth1      Link encap:Ethernet  HWaddr 02:42:AC:15:00:02
          inet addr:172.25.0.2  Bcast:0.0.0.0  Mask:255.255.0.0
          inet6 addr: fe80::42:acff:fe19:2/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:8 errors:0 dropped:0 overruns:0 frame:0
          TX packets:8 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:648 (648.0 B)  TX bytes:648 (648.0 B)

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```

在用户定义的`isolated_nw`网络上，Docker自带的DNS服务器会为网络中的其他容器提供容器名称解析。
在`container2`容器里面可以通过`container3`容器的名称ping通`container3`容器。

```bash
/ # ping -w 4 container3
PING container3 (172.25.3.3): 56 data bytes
64 bytes from 172.25.3.3: seq=0 ttl=64 time=0.070 ms
64 bytes from 172.25.3.3: seq=1 ttl=64 time=0.080 ms
64 bytes from 172.25.3.3: seq=2 ttl=64 time=0.080 ms
64 bytes from 172.25.3.3: seq=3 ttl=64 time=0.097 ms

--- container3 ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max = 0.070/0.081/0.097 ms
```

但是在`bridge`网络上并不是这样。`container2`和`container1`容器都连接到默认的`bridge`网络。
Docker并不支持此网络上的DNS解析。因为这个原因ping`container1`容器的名称会失败，正如你希望基于`/etc/hosts`文件解析一样：
（译者注：Docker老版本中并没有自定义网络更没有DNS解析，只能使用一个默认的`bridge`网络。
在这个网络中容器之间想要互相访问只能通过link配置，在`/etc/hosts`文件中定义其他容器的IP地址。
Docker为了一定程度上兼容原有默认的`bridge`网络，没有在这个`bridge`网络上加入网络相关新的特性。
所以建议读者如果不是为了兼容原有网络的话，尽量使用自定义的网络，而不是默认的`bridge`网络。）

```bash
/ # ping -w 4 container1
ping: bad address 'container1'
```

A ping using the `container1` IP address does succeed though:

```bash
/ # ping -w 4 172.17.0.2
PING 172.17.0.2 (172.17.0.2): 56 data bytes
64 bytes from 172.17.0.2: seq=0 ttl=64 time=0.095 ms
64 bytes from 172.17.0.2: seq=1 ttl=64 time=0.075 ms
64 bytes from 172.17.0.2: seq=2 ttl=64 time=0.072 ms
64 bytes from 172.17.0.2: seq=3 ttl=64 time=0.101 ms

--- 172.17.0.2 ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max = 0.072/0.085/0.101 ms
```

如果你想要使`container1`容器到`container2`容器能够正常访问。
可以使用`docker run --link`命令，这个命令可以达到通过容器名访问的效果。

使用快捷键`CTRL-p CTRL-q`退出`container2`容器。

在这个例子中，`container2`容器连接到了两个网络，因此可以与`container1`和`container3`容器。
但是`container3`和`container1`容器不在同一网络不能通信。
测试一下，现在登陆到`container3`容器，并尝试通过IP地址ping`container1`容器。

```bash
$ docker attach container3

/ # ping 172.17.0.2
PING 172.17.0.2 (172.17.0.2): 56 data bytes
^C
--- 172.17.0.2 ping statistics ---
10 packets transmitted, 0 packets received, 100% packet loss

```

运行中或非运行中容器都可以连接到网络中。 但是`docker network inspect`只显示正在运行的容器的信息。

### 在用户自定义网络中使用Link配置容器别名

在上面的例子中，`container2`容器能够在用户定义的`isolated_nw`网络中自动解析`container3`的名字。
但是在默认的`bridge`网络中这个容器名的解析没有成功。
这是为了保持对老版本Docker中默认`bridge`网络的行为的兼容[legacy link](default_network / dockerlinks.md)。

`legacy link`向默认`bridge`网络提供了4个主要功能。

* 容器名称解析
* 使用`--link=CONTAINER-NAME:ALIAS`配置容器别名
* 容器之间的连接安全(通过`--icc=false`配置隔离)
* 环境变量注入

将上述4个功能与非默认用户定义网络（如例子中的`isolated_nw`网络）进行比较。
无需任何附加配置，`docker network`提供

* 基于DNS服务的容器名称解析
* 网络中容器的自动拥有安全隔离的通信环境
* 能够动态添加到或删除于多个网络中
* 支持`--link`选项为容器提供名称别名解析

继续上面的例子，使用`--link`参数在`isolated_nw`网络中创建另一个容器`container4`。
以使用别名为同一网络中的其他容器提供额外的名称解析。

```bash
$ docker run --network=isolated_nw -itd --name=container4 --link container5:c5 busybox

01b5df970834b77a9eadbaff39051f237957bd35c4c56f11193e0594cfd5117c
```

通过配置`--link`参数，`container4`容器将能够使用别名`c5`访问`container5`容器。

请注意，在创建`container4`容器时，我们链接到一个名为`container5`的尚未创建的容器。
这是在默认`bridge`网络中的 *legacy link* 和用户定义网络中新的 *link* 功能之间的行为的差异之一。
*legacy link* 本质上是静态的，它将容器与别名硬绑定，并且不允许重新启动链接的容器。
而用户定义网络中的新的 *link* 功能在本质上是动态的，支持链接的容器重新启动，允许链接的容器出现IP地址变化。

现在让我们启动另一个名为`container5`的容器，将`container4`链接到c4。

```bash
$ docker run --network=isolated_nw -itd --name=container5 --link container4:c4 busybox

72eccf2208336f31e9e33ba327734125af00d1e1d2657878e2ee8154fbb23c7a
```

正如我们预期的，`container4`容器将能够通过`container5`的容器名称和它的别名c5访问`container5`。
`container5`容器也能够通过`container4`的容器名称及其别名c4访问`container4`。

```bash
$ docker attach container4

/ # ping -w 4 c5
PING c5 (172.25.0.5): 56 data bytes
64 bytes from 172.25.0.5: seq=0 ttl=64 time=0.070 ms
64 bytes from 172.25.0.5: seq=1 ttl=64 time=0.080 ms
64 bytes from 172.25.0.5: seq=2 ttl=64 time=0.080 ms
64 bytes from 172.25.0.5: seq=3 ttl=64 time=0.097 ms

--- c5 ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max = 0.070/0.081/0.097 ms

/ # ping -w 4 container5
PING container5 (172.25.0.5): 56 data bytes
64 bytes from 172.25.0.5: seq=0 ttl=64 time=0.070 ms
64 bytes from 172.25.0.5: seq=1 ttl=64 time=0.080 ms
64 bytes from 172.25.0.5: seq=2 ttl=64 time=0.080 ms
64 bytes from 172.25.0.5: seq=3 ttl=64 time=0.097 ms

--- container5 ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max = 0.070/0.081/0.097 ms
```

```bash
$ docker attach container5

/ # ping -w 4 c4
PING c4 (172.25.0.4): 56 data bytes
64 bytes from 172.25.0.4: seq=0 ttl=64 time=0.065 ms
64 bytes from 172.25.0.4: seq=1 ttl=64 time=0.070 ms
64 bytes from 172.25.0.4: seq=2 ttl=64 time=0.067 ms
64 bytes from 172.25.0.4: seq=3 ttl=64 time=0.082 ms

--- c4 ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max = 0.065/0.070/0.082 ms

/ # ping -w 4 container4
PING container4 (172.25.0.4): 56 data bytes
64 bytes from 172.25.0.4: seq=0 ttl=64 time=0.065 ms
64 bytes from 172.25.0.4: seq=1 ttl=64 time=0.070 ms
64 bytes from 172.25.0.4: seq=2 ttl=64 time=0.067 ms
64 bytes from 172.25.0.4: seq=3 ttl=64 time=0.082 ms

--- container4 ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max = 0.065/0.070/0.082 ms
```

与 *legacy link* 功能类似。在 *link* 网络下，使用`--link`参数别名只会影响到容器本身，别名在容器外没有任何意义。

另外，重要的是注意。即使同一个容器链接到多个网络，在某个网络中设置的别名也只在这一个网络中生效。
因此，同一个容器可以在不同网络中有不同别名。

扩展示例，让我们创建另一个名为`local_alias`的网络

```bash
$ docker network create -d bridge --subnet 172.26.0.0/24 local_alias
76b7dc932e037589e6553f59f76008e5b76fa069638cd39776b890607f567aaa
```

let us connect `container4` and `container5` to the new network `local_alias`

```
$ docker network connect --link container5:foo local_alias container4
$ docker network connect --link container4:bar local_alias container5
```

```bash
$ docker attach container4

/ # ping -w 4 foo
PING foo (172.26.0.3): 56 data bytes
64 bytes from 172.26.0.3: seq=0 ttl=64 time=0.070 ms
64 bytes from 172.26.0.3: seq=1 ttl=64 time=0.080 ms
64 bytes from 172.26.0.3: seq=2 ttl=64 time=0.080 ms
64 bytes from 172.26.0.3: seq=3 ttl=64 time=0.097 ms

--- foo ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max = 0.070/0.081/0.097 ms

/ # ping -w 4 c5
PING c5 (172.25.0.5): 56 data bytes
64 bytes from 172.25.0.5: seq=0 ttl=64 time=0.070 ms
64 bytes from 172.25.0.5: seq=1 ttl=64 time=0.080 ms
64 bytes from 172.25.0.5: seq=2 ttl=64 time=0.080 ms
64 bytes from 172.25.0.5: seq=3 ttl=64 time=0.097 ms

--- c5 ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max = 0.070/0.081/0.097 ms
```

请注意，ping在两个别名都成功了，注意访问是通过不同的网络的。
让我们通过将`container5`容器脱离`isolated_nw`网络并观察结果来结束这一节的内容。

```
$ docker network disconnect isolated_nw container5

$ docker attach container4

/ # ping -w 4 c5
ping: bad address 'c5'

/ # ping -w 4 foo
PING foo (172.26.0.3): 56 data bytes
64 bytes from 172.26.0.3: seq=0 ttl=64 time=0.070 ms
64 bytes from 172.26.0.3: seq=1 ttl=64 time=0.080 ms
64 bytes from 172.26.0.3: seq=2 ttl=64 time=0.080 ms
64 bytes from 172.26.0.3: seq=3 ttl=64 time=0.097 ms

--- foo ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max = 0.070/0.081/0.097 ms

```

总结，用户定义网络中的 *link* 功能提供 *legacy links* 的几乎所有优点，同时避免了 *legacy links* 绝大多数已知问题。

与 *legacy links* 相比，一个显着的功能缺失是支持环境变量的注入。
虽然这个功能非常有用，但是环境变量注入本质上是静态的，并且必须在容器启动时注入。
将环境变量动态注入到正在运行的容器中这个任务，我们需要很大量的工作来支持，很难以做到这一点。
因此现在我们不能在提供动态连接/断开容器到网络的`docker network`上兼容这个功能。

### 网络范围的容器别名

虽然 *link* 提供了容器范围内的名称解析。
但是网络范围的容器别名提供了更强大的功能。能在特定网络范围内，让任何其他容器都能通过别名发现另一个容器。
和服务的消费者定义的 *link* 别名不同，网络范围的容器别名需要由服务的提供者提供，向当前网络注册自己的容器别名。

继续上面的示例，在`isolated_nw`网络中创建一个带有网络别名的另一个容器。

```bash
$ docker run --network=isolated_nw -itd --name=container6 --network-alias app busybox

8ebe6767c1e0361f27433090060b33200aac054a68476c3be87ef4005eb1df17
```

```bash
$ docker attach container4

/ # ping -w 4 app
PING app (172.25.0.6): 56 data bytes
64 bytes from 172.25.0.6: seq=0 ttl=64 time=0.070 ms
64 bytes from 172.25.0.6: seq=1 ttl=64 time=0.080 ms
64 bytes from 172.25.0.6: seq=2 ttl=64 time=0.080 ms
64 bytes from 172.25.0.6: seq=3 ttl=64 time=0.097 ms

--- app ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max = 0.070/0.081/0.097 ms

/ # ping -w 4 container6
PING container5 (172.25.0.6): 56 data bytes
64 bytes from 172.25.0.6: seq=0 ttl=64 time=0.070 ms
64 bytes from 172.25.0.6: seq=1 ttl=64 time=0.080 ms
64 bytes from 172.25.0.6: seq=2 ttl=64 time=0.080 ms
64 bytes from 172.25.0.6: seq=3 ttl=64 time=0.097 ms

--- container6 ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max = 0.070/0.081/0.097 ms
```

现在让我们使用不同的网络范围别名连接`container6`容器到`local_alias`网络上。

```bash
$ docker network connect --alias scoped-app local_alias container6
```

在这个例子中，`container6`容器现在在网络`isolated_nw`中注册了别名为`app`。
而在`local_alias`网络上被别名为`scoped-app`。

让我们尝试从`container4`容器（它连接到这两个网络）和`container5`容器（只连接到`isolated_nw`网络）来访问这些别名。

```bash
$ docker attach container4

/ # ping -w 4 scoped-app
PING foo (172.26.0.5): 56 data bytes
64 bytes from 172.26.0.5: seq=0 ttl=64 time=0.070 ms
64 bytes from 172.26.0.5: seq=1 ttl=64 time=0.080 ms
64 bytes from 172.26.0.5: seq=2 ttl=64 time=0.080 ms
64 bytes from 172.26.0.5: seq=3 ttl=64 time=0.097 ms

--- foo ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max = 0.070/0.081/0.097 ms

$ docker attach container5

/ # ping -w 4 scoped-app
ping: bad address 'scoped-app'

```

正如你看到的，别名的作用域仅限于定义它的网络，因此只有那些连接到该网络的容器才能访问别名。

除了上述功能之外，多个容器可以在同一网络中共享相同的网络范围别名。
例如，让我们在`isolated_nw`网络中启动`container7`容器，并使用与`container6`相同的别名

As you can see, the alias is scoped to the network it is defined on and hence
only those containers that are connected to that network can access the alias.

In addition to the above features, multiple containers can share the same
network-scoped alias within the same network. For example, let's launch
`container7` in `isolated_nw` with the same alias as `container6`

```bash
$ docker run --network=isolated_nw -itd --name=container7 --network-alias app busybox

3138c678c123b8799f4c7cc6a0cecc595acbdfa8bf81f621834103cd4f504554
```

当多个容器共享相同的别名时，该别名将会被解析到其中一个容器（通常是定义该别名的第一个容器）。
当当前解析该别名的容器停止或从网络断开时，该别名将会被解析到下一个注册该别名的容器。

让我们从`container4`容器ping别名`app`。
并将`container6`容器停掉，来验证`container7`容器也正在解析`app`别名。

```bash
$ docker attach container4

/ # ping -w 4 app
PING app (172.25.0.6): 56 data bytes
64 bytes from 172.25.0.6: seq=0 ttl=64 time=0.070 ms
64 bytes from 172.25.0.6: seq=1 ttl=64 time=0.080 ms
64 bytes from 172.25.0.6: seq=2 ttl=64 time=0.080 ms
64 bytes from 172.25.0.6: seq=3 ttl=64 time=0.097 ms

--- app ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max = 0.070/0.081/0.097 ms

$ docker stop container6

$ docker attach container4

/ # ping -w 4 app
PING app (172.25.0.7): 56 data bytes
64 bytes from 172.25.0.7: seq=0 ttl=64 time=0.095 ms
64 bytes from 172.25.0.7: seq=1 ttl=64 time=0.075 ms
64 bytes from 172.25.0.7: seq=2 ttl=64 time=0.072 ms
64 bytes from 172.25.0.7: seq=3 ttl=64 time=0.101 ms

--- app ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max = 0.072/0.085/0.101 ms

```

## 将容器从网络中断开

你可以使用`docker network disconnect`命令断开容器与网络之间的连接。

```bash
$ docker network disconnect isolated_nw container2

$ docker inspect --format='{{json .NetworkSettings.Networks}}'  container2 | python -m json.tool

{
    "bridge": {
        "NetworkID":"7ea29fc1412292a2d7bba362f9253545fecdfa8ce9a6e37dd10ba8bee7129812",
        "EndpointID": "9e4575f7f61c0f9d69317b7a4b92eefc133347836dd83ef65deffa16b9985dc0",
        "Gateway": "172.17.0.1",
        "GlobalIPv6Address": "",
        "GlobalIPv6PrefixLen": 0,
        "IPAddress": "172.17.0.3",
        "IPPrefixLen": 16,
        "IPv6Gateway": "",
        "MacAddress": "02:42:ac:11:00:03"
    }
}


$ docker network inspect isolated_nw

[
    {
        "Name": "isolated_nw",
        "Id": "06a62f1c73c4e3107c0f555b7a5f163309827bfbbf999840166065a8f35455a8",
        "Scope": "local",
        "Driver": "bridge",
        "IPAM": {
            "Driver": "default",
            "Config": [
                {
                    "Subnet": "172.21.0.0/16",
                    "Gateway": "172.21.0.1"
                }
            ]
        },
        "Containers": {
            "467a7863c3f0277ef8e661b38427737f28099b61fa55622d6c30fb288d88c551": {
                "Name": "container3",
                "EndpointID": "dffc7ec2915af58cc827d995e6ebdc897342be0420123277103c40ae35579103",
                "MacAddress": "02:42:ac:19:03:03",
                "IPv4Address": "172.25.3.3/16",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {}
    }
]
```

一旦容器从网络中断开，它将不再能够与连接到该网络中的其他容器通信。
在这个例子中，`container2`容器不能再在`isolated_nw`网络上与`container3`容器通信。

```bash
$ docker attach container2

/ # ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:AC:11:00:03
          inet addr:172.17.0.3  Bcast:0.0.0.0  Mask:255.255.0.0
          inet6 addr: fe80::42:acff:fe11:3/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:9001  Metric:1
          RX packets:8 errors:0 dropped:0 overruns:0 frame:0
          TX packets:8 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:648 (648.0 B)  TX bytes:648 (648.0 B)

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

/ # ping container3
PING container3 (172.25.3.3): 56 data bytes
^C
--- container3 ping statistics ---
2 packets transmitted, 0 packets received, 100% packet loss
```

`container2`容器仍然与`bridge`网络保持连接

```bash
/ # ping container1
PING container1 (172.17.0.2): 56 data bytes
64 bytes from 172.17.0.2: seq=0 ttl=64 time=0.119 ms
64 bytes from 172.17.0.2: seq=1 ttl=64 time=0.174 ms
^C
--- container1 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.119/0.146/0.174 ms
/ #
```

有一些情况，例如不正常的docker daemon在多主机网络环境中重新启动，然而daemon程序又无法清除掉过时的连接端点。
当另一个新的容器用相同的连接端点名称连接到网络，这种过时的连接端点可能导致`容器已经连接到网络`的错误。
为了清理这些过时的端点，首先删除容器并强制断开端点到网络的连接（`docker network disconnect -f`）。
一旦端点被清理，容器就可以重新连接到网络。

```bash
$ docker run -d --name redis_db --network multihost redis

ERROR: Cannot start container bc0b19c089978f7845633027aa3435624ca3d12dd4f4f764b61eac4c0610f32e: container already connected to network multihost

$ docker rm -f redis_db

$ docker network disconnect -f multihost redis_db

$ docker run -d --name redis_db --network multihost redis

7d986da974aeea5e9f7aca7e510bdb216d58682faa83a9040c2f2adc0544795a
```

## 删除自定义网络

当网络中的所有容器都已经停止或断开连接时，就可以删除网络了。

```bash
$ docker network disconnect isolated_nw container3
```

```bash
$ docker network inspect isolated_nw

[
    {
        "Name": "isolated_nw",
        "Id": "06a62f1c73c4e3107c0f555b7a5f163309827bfbbf999840166065a8f35455a8",
        "Scope": "local",
        "Driver": "bridge",
        "IPAM": {
            "Driver": "default",
            "Config": [
                {
                    "Subnet": "172.21.0.0/16",
                    "Gateway": "172.21.0.1"
                }
            ]
        },
        "Containers": {},
        "Options": {},
        "Labels": {}
    }
]

$ docker network rm isolated_nw
```

列出你的所有网络，以验证`isolated_nw`网络已经被删除：

```bash
$ docker network ls

NETWORK ID          NAME                DRIVER
72314fa53006        host                host
f7ab26d71dbd        bridge              bridge
0f32e83e61ac        none                null
```

## 相关信息

* [network create](../../reference/commandline/network_create.md)
* [network inspect](../../reference/commandline/network_inspect.md)
* [network connect](../../reference/commandline/network_connect.md)
* [network disconnect](../../reference/commandline/network_disconnect.md)
* [network ls](../../reference/commandline/network_ls.md)
* [network rm](../../reference/commandline/network_rm.md)
