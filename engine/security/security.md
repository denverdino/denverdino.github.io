--- description: Review of the Docker Daemon attack surface keywords: Docker,
Docker documentation,  security redirect_from:
- /engine/articles/security/
- /security/security/ title: Docker security ---

在审视Docker安全的时候要关注4个主要的问题：

 - 内核的固有，它支持的namespace和cgroup
 - Docker daemon自身的攻击面
 - 默认配置或者用户自定义配置的容器profile的漏洞
 - 内核的安全加固特性及其和容器关系

## Kernel namespaces

Docker containers are very similar to LXC containers, and they have similar
security features. When you start a container with `docker run`, behind the
scenes Docker creates a set of namespaces and control groups for the container.

Docker容器和LXC容器很相似，具有相似的安全特性。启动容器所用的`docker
run`背后，是Docker为容器所创建的一组namespace和cgroup。

**Namespaces提供了第一种，也是最直接的一种隔离形式**:
运行在一个容器里的进程看不到，也不会被其他容器或宿主机内的进程所影响。

**每个容器拥有自己的网络栈**,
意味着容器不会获取另外一个容器的socket以及者网卡的特权访问。当然，如果宿主系统做了相应的配置，容器之间也能通过各自的网卡交互，正如你可以和外部主机交互一样。当你为容器指定公共端口亦或是使用[*links*](../userguide/networking/default_network/dockerlinks.md)，容器之间就可以通过IP通信，它们可以ping彼此，发送/接收UDP包，建立TCP连接。如果需要，也可以限制容器之间的通信。从网络架构的角度，同一台宿主机上的Docker容器接入了网桥设备，恰恰就像接入了以太网交换机的物理机。

实现内核namespace和私有网络的代码有多成熟呢？内核namespace[在内核版本2.6.15和2.6.26之间](http://man7.org/linux/man-pages/man7/namespaces.7.html)引入，意味着自从2008年7月（2.6.26的发布时间），namespace的代码就在大量的生产系统里接收考验。另外，namespace的灵感和设计比代码还要早。Namespace实际上重新实现了[OpenVZ](
http://en.wikipedia.org/wiki/OpenVZ)的特性，以合并到内核主线里。OpenVZ在2005年就发布了，所以namespace设计和实现都已经很成熟。

## Control groups

Linux容器的另外一个关键组件是Control Group，实现资源计数和限制。Control
Group提供了很多有用的计量，同时确保每个容器公平的共享内存、CPU、磁盘IO，更重要的是，单个容器不会因为耗尽某项资源导致系统挂掉。

它们的存在不是为了阻止容器访问其他容器内进程和数据，而是避免系统受到DoS攻击。在诸如公有云和私有云PaaS这类多租户平台上尤其重要，在某些程序存在恶意行为的情况下也能保证系统的正常运行。

Control Group也早已存在：代码起始于2006，最早合并到2.6.24版本的内核。

## Docker daemon的攻击面

使用Docker运行容器的隐含前提是你得运行Docker
daemon。目前daemon需要`root`权限，所以你得关注一些重要的细节。

首先，**只允许受信任的用户控制Docker
daemon**，很容易从Docker强大的功能里得出这个结论。尤其是Docker允许你在宿主机和容器之间共享目录，还可以不限制容器的访问权限。你甚至可以把宿主机`/`目录挂载到容器的`/host`目录，容器内可以无任何限制的修改宿主机文件系统。这点和虚拟化系统所支持的文件系统资源共享很相似，没有什么能阻止你给虚拟机共享根文件系统。

一个很重要的安全提示：如果你通过Web服务器用API启动Docker容器，尤其要注意参数校验，以确保不能传入精心构造的参数来创建任意容器。

因此，在Docker 0.5.2时REST API端点从127.0.0.1改成了使用Unix socket。
后者允许使用传统的Unix权限检查限制对控制socket的访问。

如果真的需要，你也可以通过HTTP暴露REST
API。不过在这样做的时候，要意识到前面提到的安全风险，并确保只暴露到受信任的网络或者VPN里，或者通过例如`stunnel`和客户端SSL证书保护，参考[HTTPS和certificates](https.md)

daemon也有潜在的缺陷，比如通过`docker load`从磁盘加载镜像，或者通过`docker
pull`从网络加载镜像。在Linux/Unix平台上，Docker
1.3.2，镜像在chrooted之后的子进程里解压，这是安全加固的第一步。在Docker
1.10.0，所有镜像可以根据内容摘要存储和访问，限制了可能存在的修改现有镜像攻击。

最终，期望做到限制Docker
daemon的特权，把一些操作代理到具有良好审计的子进程里，每个自己成有自己有限的Linux
capabilities：虚拟网络设置，文件系统管理等等。这种方式更像是让Docker
Engine的每一块运行在一个容器里。

如果你在服务器上运行Docker，建议在服务器上只运行Docker，其他服务移到容器里，通过Docker控制。当然，可以保留你喜欢的管理工具（可能至少有SSH服务），还有监控和管理进程，比如NRPE和collectd

## Linux内核capabilities

默认情况下，Docker以有限的capabilities集启动容器，什么意思呢？

Capabilities把原来`root/非root`的二元划分改成了细粒度的访问控制系统。只需要绑定1024一下端口的进程不需要用root运行：只需要授予`net_bind_service`
capability。还有很多其他的capabilities，对应大多数通常需要root权限的地方。

这对容器安全意义重大，我们来看看为什么！

服务器上（物理机或者虚拟机）上需要以root权限运行大量进程，典型情况下包括SSH、cron、syslogd、硬件管理工具(例如加载模块)、网络配置工具（例如处理DHCP，WPA或者VPN）等等。容器非常不一样，因为几乎全部任务都由容器相关的工具处理：

 - SSH访问运行在Docker宿主机上，被单一服务管理
 - `cron` 必要的时候作为用户进程运行，服务于需要定时的程序，而不是作为系统层工具
 - 日志管理也可以交给Docker管理，或者采用第三方服务，比如Loggly或者Splunk
 - 硬件管理很独立，你可以在容器里运行`udevd`或者等价的服务。
 - 网络管理发生在容器之外，强制尽可能分离关注点。容器永远不要执行`ifconfig`,
   `route`或者`ip`命令（除非容器专门用于特殊情况，比如路由，防火墙）

这意味着多数情况下，容器**根本**不需要""真正的"root权限"，因此，容器可以在削减capability情况下运行。也意味着容器内的root比真正的root权限少。例如可能存在：

 - 拒绝所有“mount”操作。
 - 拒绝访问原始socket（防止伪造数据包）
 - 拒绝一些文件系统操作，如创建新设备节点，改变文件所有者，或者修改属性
 - 拒绝加载模块
 - 其他。。。

这意味着即使攻击者在容器里提权成root，也很难做严重的破坏或者逃逸到宿主机。

这不影响常规web应用，但是恶意用户会发现他们的武器效果严重缩水！默认情况下Docker只保留了[]如下的capabilities](https://github.com/docker/docker/blob/master/oci/defaults_linux.go#L62-L77)。全部capabilities列表在[这里](http://man7.org/linux/man-pages/man7/capabilities.7.html)

运行Docker容器的一个主要风险是，给容器的默认capabilities集合以及mounts可能导致不完全的隔离。

Docker支持添加或者移除capabilities，允许使用非默认profile，移除capabilities可以让容器更安全，添加capabilities则降低了安全性。最佳实践是移除所有capabilities，只保留真正需要的。

## 其他内核安全特性

Capabilities只是现代Linux内核众多安全特性中的一个，也可以配置Docker使用现有的众所周知的安全系统：比如TOMOYO，AppArmor，SELinux，GRSEC等

因为Docker目前只启用了capabilities，不影响其他组件，所以可以用很多其他方式加固Docker宿主机。下面是几个例子：
 - 你可以运行patch了GRSEC和PAX的内核。它们增加了很多安全校验，包括编译时和运行时。它们还可以抵御很多攻击，多亏有了类似地址随机化技术。不需要特定于Docker的配置，这些安全特性都是系统范围的，和容器无关。
 - 如果你的发行版包含Docker容器安全模型的模板，你可以立即使用。例如，我们提供了AppArmor和SELinux策略的模板。这些模板提供了额外的安全。
 - 你可以用你喜欢的访问控制机制定义自己的策略。

很多第三方工具可以增强Docker容器，例如特殊的网络拓扑或者共享文件系统，你可以找些工具加固Docker容器安全性而不影响Docker核心。

Docker
1.10直接支持用户命名空间，这个特性允许容器内的root用户映射成外部的非uid-0用户，可以降低容器逃逸造成的危害。可以用，但默认没有启用。 参考[daemon命令](../reference/commandline/dockerd.md#daemon-user-namespace-options) 以获取这个特性的更多信息.  更多关于Docker里用户命名空间实现的信息参考 <a href="https://integratedcode.us/2015/10/13/user-namespaces-have-arrived-in-docker/" target="_blank">这个</a>.

## 结论

Docker容器默认就很安全，尤其是你注意在容器里用非特权用户运行进程。

你可以通过启用AppArmor，SELinux，GRSEC或者你喜欢的加固方案，提供额外的安全层。

最后但同样重要的，如果你在其他容器系统里看到了有趣的安全特性，这些简单的内核特性
可能已经在Docker里实现了。我们欢迎用户提交issues，pull requests，以及通过邮件列
表沟通。

## 相关信息

* [使用可信镜像](../security/trust/index.md)
* [Docker的Seccomp安全profiles](../security/seccomp.md)
* [Docker的AppArmor安全profiles](../security/apparmor.md)
* [容器安全(2014)](https://medium.com/@ewindisch/on-the-security-of-containers-2c60ffe25a9e)
* [Docker swarm mode overlay网络安全模型](../userguide/networking/overlay-security-model.md)

