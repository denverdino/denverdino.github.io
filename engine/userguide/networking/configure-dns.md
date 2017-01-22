---
description: Learn how to configure DNS in user-defined networks
keywords: docker, DNS, network
title: Embedded DNS server in user-defined networks
---

本节中的信息，涵盖了在用户定义网络中对容器嵌入式DNS服务器的一些操作。
连接到用户定义网络的容器的DNS服务与连接到`default bridge`网络提供的host绑定不同。

> **Note**: 为了保持向后兼容，在`default bridge`网络中的DNS配置保留了原有的行为。
> 有关`default bridge`网络中的DNS配置的详细信息，请参阅[DNS in default bridge network](default_network/configure-dns.md)。

从Docker 1.10开始，docker daemon实现了一个嵌入式DNS服务器，
它为使用了有效的`name`或`net-alias`或`link`参数创建的任何容器，提供内置的服务发现功能。
了解Docker如何管理容器自带的DNS服务，可以帮助你从当前的Docker版本升级到新的版本。
所以你不应该假设`/etc/hosts`，`/etc/resolv.conf`这样的文件在容器内部被管理。
放弃使用这些文件，改换成使用下面这些Docker选项。

影响容器DNS服务的各种选项。

<table>
  <tr>
    <td>
    <p>
    <code>--name=CONTAINER-NAME</code>
    </p>
    </td>
    <td>
    <p>
     在用户定义的docker网络中，使用 <code>-name</code> 配置的容器名称。
     内置DNS服务器会维护容器名称到其IP地址（容器连接的网络上）之间的映射。
    </p>
    </td>
  </tr>
  <tr>
    <td>
    <p>
    <code>--network-alias=ALIAS</code>
    </p>
    </td>
    <td>
    <p>
     除了如上所述的<code>-name</code>，容器被其配置的一个或多个<code>--network-alias</code>（或在<code>docker network connect</code>命令中使用<code>-alias</code>参数）。
     内置DNS服务器维护所有容器别名到其指定用户定义网络上的IP地址之间的映射。
     通过使用<code>docker network connect</code>命令中的<code>-alias</code>选项，容器可以在不同的网络中具有不同的别名。
    </p>
    </td>
  </tr>
  <tr>
    <td>
    <p>
    <code>--link=CONTAINER_NAME:ALIAS</code>
    </p>
    </td>
    <td>
    <p>
      当你使用<code>run</code>命令启动容器时，这个参数将给内置DNS一条记录。
      该记录将解析<code>ALIAS</code>指向由<code>CONTAINER_NAME</code>标识的容器的IP地址。
      当使用<code>-link</code>时，内置DNS将保证本地服务查找的结果指向使用<code>-link</code>的容器上。
      这使得新容器中的进程不必知道其他容器的名称或IP，就能够连接到其他容器。
    </p>
    </td>
  </tr>
  <tr>
    <td><p>
    <code>--dns=[IP_ADDRESS...]</code>
    </p></td>
    <td><p>
     如果内置DNS服务器无法解析容器的名称解析请求，内置DNS服务器将使用通过<code>-dns</code>选项传递的IP地址转发DNS查询。
     这些<code>-dns</code>中设置的IP地址由内置DNS服务器管理，并且不会在容器的<code>/etc/resolv.conf</code>文件中更新。
  </tr>
  <tr>
    <td><p>
    <code>--dns-search=DOMAIN...</code>
    </p></td>
    <td><p>
    当一个独立的主机名被搜索时，设置这个主机名的搜索域名。
    <code>--dns-search</code>选项中设置的域名将由内置DNS服务器管理，不会在容器的<code>/etc/resolv.conf</code>文件中更新。
    例如，当容器进程尝试访问<code>host</code>并且搜索域名<code>example.com</code>被设置时。
    DNS服务不仅会尝试查找<code>host</code>主机，也会尝试查找<code>host.example.com</code>。
    </p>
    </td>
  </tr>
  <tr>
    <td><p>
    <code>--dns-opt=OPTION...</code>
    </p></td>
    <td><p>
      设置DNS服务的使用选项。这些选项由内置DNS服务器管理，并且不会在容器的<code>/etc/resolv.conf</code>文件中更新。
    </p>
    <p>
    查看<code>resolv.conf</code>的文档，可以了解其可用参数
    </p></td>
  </tr>
</table>


在没有使用`--dns=IP_ADDRESS...`，`--dns-search=DOMAIN...`或`--dns-opt=OPTION ...`选项的情况下，
Docker会使用宿主机的`/etc/resolv.conf`（`docker daemon`运行的地方）。
在这样做时，`docker daemon`会从主机的原始文件中过滤掉所有localhost IP地址的域名解析条目。

过滤是必要的，因为主机上的所有本地主机地址都是无法从容器的网络访问的。
在这个过滤之后，如果容器的`/etc/resolv.conf`文件中没有其他的域名解析条目，
docker daemon会将公开的Google DNS名称服务器（8.8.8.8和8.8.4.4）添加到容器的DNS配置中。
如果守护程序启用了IPv6，则还会添加公共IPv6 DNS DNS服务器（2001:4860:4860::8888和2001:4860:4860::8844）。

> **Note**：如果需要容器访问主机的本地主机解析，
> 则必须修改主机上的DNS服务，以便访问可以容器中访问的非localhost地址。
