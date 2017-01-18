---
description: Learn how to configure DNS in Docker
keywords: docker, bridge, docker0, network
title: Configure container DNS
---

本节中的信息说明如何在Docker默认网桥中配置容器DNS。 这是在安装Docker时自动创建的以`bridge`命名的网桥。

> **注意**：[Docker网络功能](../index.md)允许您创建除默认网桥网络之外的用户自定义网络。 有关用户定义网络中DNS配置的更多信息，请参阅[Docker配置DNS](../configure-dns.md)部分。

Docker如何为每个容器提供主机名和DNS配置，而无需构建具有写入主机名的自定义镜像？它的诀窍是将三个关键的`/etc`文件覆盖容器中的虚拟文件，它们可以写入新的信息。 您可以通过在容器中运行mount来看到这一点：
```
root@f38c87f2a42d:/# mount

...
/dev/disk/by-uuid/1fec...ebdf on /etc/hostname type ext4 ...
/dev/disk/by-uuid/1fec...ebdf on /etc/hosts type ext4 ...
/dev/disk/by-uuid/1fec...ebdf on /etc/resolv.conf type ext4 ...
...
```

这种实现允许Docker做一些聪明的事情，比如在主机通过DHCP接收新配置时，可以对所有容器的`resolv.conf`进行更新。 关于Docker如何在容器中维护这些文件的确切细节在不同版本的Docker中可能有变化，因此您应该不去使用这些文件，而使用以下Docker选项来代替。

四个不同的选项对容器域名服务的影响。
<table>
  <tr>
    <td>
    <p>
    <code>-h HOSTNAME</code> or <code>--hostname=HOSTNAME</code>
    </p>
    </td>
    <td>
    <p>
    设置容器自身的主机名。这被写入 <code>/etc/hostname</code>，作为容器面向主机的IP地址的名称写入 <code>/etc/hosts</code>，并且是容器中的 <code>/bin/bash</code> 将在其提示符内显示的名称。但是主机名不容易从容器外面看到。它不会出现在 <code>docker ps</code>中，也不会出现在任何其他容器的<code>/etc/hosts</code>文件中。  
    </p>
    </td>
  </tr>
  <tr>
    <td>
    <p>
    <code>--link=CONTAINER_NAME</code> or <code>ID:ALIAS</code>
    </p>
    </td>
    <td>
    <p>
    在运行容器时使用此选项会为新容器的 <code>/etc/hosts</code>附加一个名为<code>ALIAS</code>的额外条目，该条目指向由 <code>CONTAINER_NAME_or_ID</code> 标识的容器的IP地址。 这允许新容器内的进程连接到主机名<code>ALIAS</code>，而不必知道其IP。下面更详细地讨论了 <code>--link=</code> 选项。因为Docker可以在重新启动时为链接的容器分配不同的IP地址，Docker会更新接收容器的 <code>/etc/hosts</code> 文件中的ALIAS条目。  
</p>
    </td>
  </tr>
  <tr>
    <td><p>
    <code>--dns=IP_ADDRESS...</code>
    </p></td>
    <td><p>
将作为服务器行添加的IP地址设置为容器的 <code>/etc/resolv.conf</code> 文件。容器中的进程在查找不在 <code>/etc/hosts</code> 中的主机名时，将连接到这些IP地址的53端口，以查找域名解析服务。
</p></td>
  </tr>
  <tr>
    <td><p>
    <code>--dns-search=DOMAIN...</code>
    </p></td>
    <td><p>
    设置容器在使用未限定主机名时搜索的域名，方法是将搜索行写入容器的 <code>/etc/resolv.conf</code>。当容器进程尝试访问 <code>host</code> 并搜索域名 <code>example.com</code> 时，DNS逻辑将不仅查找主机，而且查找 <code>host.example.com</code>。
    </p>
    <p>
    使用 <code>--dns-search=</code> 。 如果您不想设置搜索域。  
    </p>
    </td>
  </tr>
  <tr>
    <td><p>
    <code>--dns-opt=OPTION...</code>
    </p></td>
    <td><p>
    设置DNS解析器使用的选项，方法是在容器的 <code>/etc/resolv.conf</code> 中写入一个 <code>options</code> 行。  
    </p>
    <p>
    有关有效选项的列表，请参见 <code>resolv.conf</code> 的文档  
    </p></td>
  </tr>
  <tr>
    <td><p></p></td>
    <td><p></p></td>
  </tr>
</table>


关于DNS设置，如果没有`--dns=IP_ADDRESS...`，`--dns-search=DOMAIN...`或`--dns-opt=OPTION...`选项，Docker会使每个容器的`/etc/resolv.conf`看起来像主机的`/etc/resolv.conf`（Docker守护程序运行的地方）。当创建容器的`/etc/resolv.conf`时，守护进程从主机的原始文件中过滤掉所有localhost IP地址`nameserver`条目。

过滤是必要的，因为主机上的所有本地主机地址都无法从容器的网络访问。 过滤后，如果容器的`/etc/resolv.conf`文件中没有更多的名称服务器条目，则守护程序会将Google的公共DNS服务器（8.8.8.8和8.8.4.4）添加到容器的DNS配置中。如果守护程序启用了IPv6，则还会添加公共IPv6 DNS服务器（200:1h4860:4860::8888和2001:4860:4860::8844）。

> **注意**：如果需要访问主机的本地主机解析器，则必须在主机上修改DNS服务，以侦听可从容器中访问的非本地主机地址。

您可能想知道当主机的`/etc/resolv.conf`文件更改时会发生什么。docker守护程序具有活动的文件更改通知程序，它将监视对主机DNS配置的更改。

> **注意**：文件更改通知程序依赖于Linux内核的inotify功能。 因为此功能当前与overlay文件系统驱动程序不兼容，所以使用“overlay”的Docker守护程序将无法利用`/etc/resolv.conf`自动更新功能。

当主机文件更改时，所有已停止的具有与主机匹配的`resolv.conf`的容器将立即更新到此最新的主机配置。 当主机配置更改时，因为缺少一个确保容器运行时`resolv.conf`文件的原子写入的工具，所以正在运行的容器将需要停止并启动去收集主机更改。 如果容器的`resolv.conf`在使用默认配置启动后已被编辑，则不会尝试替换，因为它将覆盖容器执行的更改。如果选项（`--dns`，`--dns-search`或`--dns-opt`）已被用于修改默认主机配置，则用更新的主机的`/etc/resolv.conf`替换也不会发生。

> **注意**：对于在Docker 1.5.0中实现`/etc/resolv.conf`更新功能之前创建的容器：当主机`resolv.conf`文件更改时，这些容器将 **不会** 收到更新。 只有使用Docker 1.5.0及更高版本创建的容器才会使用此自动更新功能。
