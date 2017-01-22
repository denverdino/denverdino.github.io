---
description: Use the routing mesh to publish services externally to a swarm
keywords:
- guide
- swarm mode
- swarm
- network
- ingress
- routing mesh
title: Use swarm mode routing mesh
---
Docker引擎Swarm模式让您可以很容器的对外发布服务端口，使服务再集群外可用。所有节点都会加入**路由网格**。路由网格让集群中的每个节点能够接受来自集群中任何服务发布端口上的连接，即使节点上没有任何任务正在运行。路由网络将所有传入请求路由到可用节点上的已发布端口。

为了在集群中使用路由网格，您需要在集群节点之间打开以下端口，然后启用swarm模式：

* 端口`7946` TCP/UDP用于容器网络发现。
* 端口`4789` UDP用于容器入口网络。

您还必须打开集群节点和需要访问端口的任何外部资源（例如外部负载均衡器）之间的已发布端口。

## 发布服务端口

当你创建服务时，使用 `--publish` 标签来发布端口:

```bash
$ docker service create \
  --name <SERVICE-NAME> \
  --publish <PUBLISHED-PORT>:<TARGET-PORT> \
  <IMAGE>
```

标签`<TARGET-PORT>`： 容器监听端口.
标签`<PUBLISHED-PORT>`：Swarm对外发布端口，使服务对外可用。

例如：下面命令向swarm中节点发布8080端口，其对应容器中nginx服务的80端口；

```bash
$ docker service create \
  --name my-web \
  --publish 8080:80 \
  --replicas 2 \
  nginx
```
当您在任何节点上访问端口8080时，集群负载均衡器会将您的请求路由到可用容器。

路由网络在节点的任何IP地址上监听发布的端口。对于外部可路由的IP地址，端口对外部节点是可用的。对于所有其他IP地址，只能从主机内部中访问。

![service ingress image](images/ingress-routing-mesh.png)

您可以使用以下命令为现有服务发布端口：

```bash
$ docker service update \
  --publish-add <PUBLISHED-PORT>:<TARGET-PORT> \
  <SERVICE>
```

您可以使用 `docker service inspect` 来查看服务的发布端口。例如：

```bash
{% raw %}
$ docker service inspect --format="{{json .Endpoint.Spec.Ports}}" my-web

[{"Protocol":"tcp","TargetPort":80,"PublishedPort":8080}]
{% endraw %}
```

输出显示容器目标端口`<TARGET-PORT>`和节点服务所监听的发布端口`<PUBLISHED-PORT>`。

### 仅发布TCP端口或UDP端口

默认情况下，您发布的是一个TCP端口。您也可以专门发布UDP端口。当发布TCP和UDP端口时，Docker 1.12.2和更早版本需要您为TCP端口添加后缀`/tcp`。否则它是可选的。


#### 只发布TCP端口

以下两个命令是等效的。

```bash
$ docker service create --name dns-cache -p 53:53 dns-cache

$ docker service create --name dns-cache -p 53:53/tcp dns-cache
```

#### 同时发布TCP、UDP端口

```bash
$ docker service create --name dns-cache -p 53:53/tcp -p 53:53/udp dns-cache
```

#### 只发布UDP端口

```bash
$ docker service create --name dns-cache -p 53:53/udp dns-cache
```

## 配置外部负载均衡
您可以配置外部负载均衡将请求路由到集群内的服务。例如，您可以配置[HAProxy](http://www.haproxy.org)来均衡请求到nginx服务。

![使用外部负载均衡映像插入](images/ingress-lb.png)

在这种情况下，端口8080必须在负载均衡和集群中的节点之间打开。集群节点可以在代理服务器可访问，但不对外公开访问的专用网络上。

您可以将负载均衡配置为集群中的每个节点相互均衡，即使节点上没有任务也是如此。例如，您可以在`/ etc / haproxy / haproxy.cfg`中配置以下HAProxy配置：

```bash
global
        log /dev/log    local0
        log /dev/log    local1 notice
...snip...

# Configure HAProxy to listen on port 80
frontend http_front
   bind *:80
   stats uri /haproxy?stats
   default_backend http_back

# Configure HAProxy to route requests to swarm nodes on port 8080
backend http_back
   balance roundrobin
   server node1 192.168.99.100:8080 check
   server node2 192.168.99.101:8080 check
   server node3 192.168.99.102:8080 check
```
当您在端口80上访问HAProxy负载均衡器时，它会将请求转发到集群中的节点上。集群路由网络将请求路由到可用的容器。如果由于任何原因swarm调度器将任务分到不同的节点，则不需要重新配置负载均衡器。

您可以配置任何类型的负载均衡器以将请求路由到集群各个节点。要了解有关HAProxy的更多信息，请参阅[HAProxy文档](https://cbonte.github.io/haproxy-dconv/)。


## 更多

* [swarm中部署服务](services.md)
