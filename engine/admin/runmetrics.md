---
description: Measure the behavior of running containers
keywords: docker, metrics, CPU, memory, disk, IO, run,  runtime, stats
redirect_from:
- /engine/articles/run_metrics
- /engine/articles/runmetrics
title: Runtime metrics
---

## Docker 状态

你可以使用`docker stats`命令来监控一个容器运行时度量。 该命令支持CPU，内存使用，内存限制，
和网络IO指标。

下面是从'docker stats`命令输出的示例

```bash
$ docker stats redis1 redis2

CONTAINER           CPU %               MEM USAGE / LIMIT     MEM %               NET I/O             BLOCK I/O
redis1              0.07%               796 KB / 64 MB        1.21%               788 B / 648 B       3.568 MB / 512 KB
redis2              0.07%               2.746 MB / 64 MB      4.29%               1.266 KB / 648 B    12.4 MB / 0 B
```

[docker stats](../reference/commandline/stats.md) 参考页有关于`docker stats`命令的更多详细信息。

## 控制组

Linux容器依赖[控制组](https://www.kernel.org/doc/Documentation/cgroup-v1/cgroups.txt)，它不仅跟踪流程组，而且公开有关的度量CPU，内存和块I / O使用。 您可以访问这些指数获取网络使用指标。 这是“纯”LXC相关容器，以及Docker容器。

控制组通过伪文件系统公开。 在最近发行版里面，你应该在`/sys/fs/cgroup`下找到这个文件系统。在
那个目录下，你会看到多个子目录，称为设备，blkio等; 每个子目录实际上对应一个不同的cgroup层次结构。

在旧系统上，控制组可能挂载在`/cgroup`上，而不是不同的层次结构。 在这种情况下，不是看到子目录，
您将在该目录中看到一堆文件，可能还会看到一些目录对应现有容器。

要确定控件组的安装位置，您可以运行：

```bash
$ grep cgroup /proc/mounts
```

## 枚举   cgroups

您可以查看`/proc/cgroups`以查看不同的控制组子系统， 它们所属的层次结构，以及它们包含多少个组。

你也可以看一下`/proc/<pid>/cgroup`来查看一个进程属于哪个控制组。 控制组将显示为相对于的根路径以及层次安装点; 例如，`/`表示“此进程尚未分配给一个特定的组“，而`/lxc/pumpkin`意味着这个进程可能是一个名为`pumpkin`的容器的成员。

## 找到给定容器的   cgroup

对于每个容器，将在每个层次结构中创建一个cgroup。 旧系统使用旧版本的LXC用户工具，cgroup将是容器的名称。 在最新版本的LXC工具中，cgroup将是 `lxc/<container_name>.`

对于使用cgroups的Docker容器，容器名称将是完整的ID或容器的长ID。 如果容器显示为ae836c95b4c3
在`docker ps`中，它的长ID可能是这样
`ae836c95b4c3c9e9179e0e91015512da89fdec91612f63cebae57df9a5444c79`。 您可以
用`docker inspect`或`docker ps --no-trunc`查找。

将所有内容放在一起来看看Docker的内存指标
容器，可以查看`/sys/fs/cgroup/memory/docker/<longid>/`。

## 来自cgroups的度量标准：内存，CPU，块I / O

对于每个子系统（内存，CPU和块I / O），您将找到一个或更多包含统计信息的伪文件。

### 内存指标: `memory.stat`

内存指标位于“内存”cgroup中。 注意内存控制组增加了一点开销，因为它非常细粒度统计了计算主机上的内存使用情况。 因此，许多发行版选择不默认启用它。 一般来说，为了使它，你所做的是添加一些内核命令行参数：`cgroup_enable = memory swapaccount = 1`。

这些度量在伪文件`memory.stat`中。它大概长这样子：

    cache 11492564992
    rss 1930993664
    mapped_file 306728960
    pgpgin 406632648
    pgpgout 403355412
    swap 0
    pgfault 728281223
    pgmajfault 1724
    inactive_anon 46608384
    active_anon 1884520448
    inactive_file 7003344896
    active_file 4489052160
    unevictable 32768
    hierarchical_memory_limit 9223372036854775807
    hierarchical_memsw_limit 9223372036854775807
    total_cache 11492564992
    total_rss 1930993664
    total_mapped_file 306728960
    total_pgpgin 406632648
    total_pgpgout 403355412
    total_swap 0
    total_pgfault 728281223
    total_pgmajfault 1724
    total_inactive_anon 46608384
    total_active_anon 1884520448
    total_inactive_file 7003344896
    total_active_file 4489052160
    total_unevictable 32768


前半部分（没有`total_`前缀）包含相关的统计信息到cgroup中的进程，不包括子cgroups。 下半部分
（使用`total_`前缀）也包括子cgroup。

一些度量是“估计的”，即，可以增加或减少的度量值（例如，交换，由cgroup的成员使用的交换空间的量）。其他一些是“计数器”，即只能上升的值，因为它们表示特定事件的发生（例如，pgfault，其
表示自创建以来发生的页面错误的数量cgroup; 这个数字永远不会减少）。

<style>table tr > td:first-child { white-space: nowrap;}</style>

指标                                | 描述
--------------------------------------|-----------------------------------------------------------
**cache**                             | 此控制组的进程使用的内存量，可以精确地与块设备上的块相关联。 当您从磁盘上的文件读取和写入时，此数量将增加。 如果你使用“常规”I / O（`open`，`read`，`write` 系统调用）以及映射文件（用`mmap`），就会出现这种情况。 它也考虑了`tmpfs` mounts使用的内存，虽然原因不清楚。
**rss**                               | 不与磁盘上的任何内容对应的内存量：堆栈，堆和匿名内存映射。
**mapped_file**                       | 指示控制组中的进程映射的内存量。 它不给你关于*使用多少*内存的信息; 它宁愿告诉你如何使用它。
**pgfault**, **pgmajfault**           | 指示cgroup的进程分别触发“页错误”和“主要故障”的次数。当进程访问其不存在或受保护的虚拟存储器空间的一部分时，发生页错误。前者可能发生，如果过程是错误的，并试图访问一个无效的地址（它会被发送一个`SIGSEGV`信号，通常会使用著名的`Segmentation `消息杀死）。后者可能发生在进程从已交换出的内存区域读取或对应于映射文件时：在这种情况下，内核将从磁盘加载页面，并让CPU完成内存访问。当进程写入写时复制存储区时也会发生这种情况：同样，内核将抢占进程，复制内存页，并恢复对进程自己的页副本的写操作。 “主要”故障发生时内核实际上必须从磁盘读取数据。当它只需要复制一个现有的页面或分配一个空页面时，它是一个常规（或“次要”）故障。
**swap**                              | 此cgroup中的进程当前使用的交换量。
**active_anon**, **inactive_anon**    | 已被标识的*匿名*内存的量分为内核*活动*和*非活动*。 “匿名”内存是*不*链接到磁盘页的内存。 换句话说，这相当于上面描述的rss计数器。 实际上，rss计数器的定义是** active_anon ** + ** inactive_anon ** - ** tmpfs **（其中tmpfs是由此控制组装载的`tmpfs`文件系统使用的内存量）。 现在，“活动”和“非活动”有什么区别？ 页面最初是“活动”; 并且以规则的间隔，内核扫描存储器，并且将一些页面标记为“非活动”。 每当它们被再次访问时，它们立即被重新标记为“活动”。 当内核几乎耗尽内存，时间来到磁盘交换时，内核将交换“非活动”页面。
**active_file**, **inactive_file**    | 缓存内存，具有* active *和* inactive *类似于上面的* anon *内存。 确切的公式是** cache ** = ** active_file ** + ** inactive_file ** + ** tmpfs **。 内核在活动和非活动集之间移动内存页所使用的确切规则与用于匿名内存的内核页不同，但一般原则是相同的。 请注意，当内核需要回收内存时，从此池中回收一个干净（=未修改）页面是比较便宜的，因为它可以立即被回收（而匿名页面和脏/修改的页面必须首先写入磁盘） 。
**unevictable**                       | 无法回收的内存量; 一般来说，它将占用已经被`mlock`锁定的内存。 它通常由加密框架使用，以确保密钥和其他敏感材料不会被交换到磁盘。
**memory_limit**, **memsw_limit**     | 这些不是真正的度量，而是一个提醒的限制应用于此的cgroup。 第一个表示此控制组的进程可以使用的物理内存的最大量; 第二个表示RAM +交换的最大量。

考虑页面缓存中的内存是非常复杂的。 如果两个
不同控制组中的进程读取同一个文件（最终依靠磁盘上的相同块），相应内存将在控制组之间分割。 这是很好，但这也意味着当cgroup终止时，它可以增加另一个cgroup的内存使用，因为它们不会再分割那些内存页。

### CPU 指标: `cpuacct.stat`

现在我们已经涵盖了内存指标，其他一切都会非常简单的比较。 CPU度量标准可以在`cpuacct`控制器中找到。

对于每个容器，你会发现一个伪文件`cpuacct.stat`，包含由容器的进程累积的CPU使用，在`user`和`system`时间之间分解。 如果你不熟悉有区别，`user`是进程直接控制CPU的时间（即执行进程代码,`system`是CPU执行系统调用的时间。

那些时间以1/100秒的刻度表示。 其实，它们在“用户jiffies”中表示。 有`USER_HZ`*“jiffies”*每秒，而在x86系统上，`USER_HZ`是100.这用来映射到每秒调度程序数“ticks”; 但随着更高的出现
频率调度，以及[无tick内](http://lwn.net/Articles/549580/)，内核滴答的数量不再相关。 它无处不在，这主要是遗产和兼容性原因。

### Block I/O 指标

块I / O被记录在`blkio'控制器中。不同的度量分散在不同的文件中。 虽然你可以
在[blkio-controller](https://www.kernel.org/doc/Documentation/cgroup-v1/blkio-controller.txt)中找到详细的细节文件在内核文档中，这里是最简短的列表相关：


指标                      | 描述
----------------------------|-----------------------------------------------------------
**blkio.sectors**           |包含由cgroup的进程成员读取和写入的512字节扇区数，逐个设备。 读取和写入合并在单个计数器中。
**blkio.io_service_bytes**  | 表示cgroup读写的字节数。 每个器件有4个计数器，因为对于每个器件，它区分同步与异步I / O以及读取与写入。
**blkio.io_serviced**       | 执行的I / O操作数，而不考虑它们的大小。 它还有每个设备4个计数器。
**blkio.io_queued**         | 表示当前为此cgroup排队的I / O操作数。 换句话说，如果cgroup没有做任何I / O，这将是零。 注意，相反的是不是真的。 换句话说，如果没有I / O队列，它不意味着cgroup空闲（I / O方式）。 它可以在其他静态设备上进行纯同步读取，因此能够立即处理它们，而不排队。 此外，尽管确定哪个cgroup正在强调I / O子系统是有帮助的，但请记住它是一个相对量。 即使进程组没有执行更多的I / O，其队列大小也可能增加，因为设备负载由于其他设备而增加。

## Network 指标

网络度量不会直接由控制组公开。 这里有一个好的解释：网络接口存在于上下文中的*网络命名空间*。 内核可能累积关于由一组进程发送和接收的包和字节的度量，但是那些指标将不是非常有用。 您需要每个接口的指标。因为流量在本地“lo”上发生(接口没有真正计数）。 但是因为进程在一个cgroup中可以属于多个网络命名空间，那些度量会更难解释：多个网络命名空间意味着多个`lo`接口，可能多个`eth0`接口等; 所以这就是为什么没有简单的方法收集网络指标与控制组。

相反，我们可以从其他来源收集网络指标：

### IPtables

IPtables（或者，更确切地说，iptables是netfilter框架一个接口）可以做一些详细的计费。

例如，您可以设置一个规则来统计出站HTTPWeb服务器上的流量：

```bash
$ iptables -I OUTPUT -p tcp --sport 80
```

没有`-j`或`-g`标志，所以规则将只计数匹配的数据包并转到以下规则。

稍后，可以使用以下命令检查计数器的值：

```bash
$ iptables -nxvL OUTPUT
```

从技术上讲，`-n'不是必需的，但它会的防止iptables进行DNS反向查找，这可能是在这种情况下无用。

计数器包括数据包和字节。 如果您要为其设置容器流量统计，你可以执行一个`for`循环添加两个`iptables`规则容器IP地址（每个方向一个），在`FORWARD`中。 这将只计算流经NAT的流量
层; 您还必须添加通过用户界的流量代理。

然后，您需要定期检查这些计数器。 如果你碰巧使用`collectd`，有一个[nice插件](https://collectd.org/wiki/index.php/Table_of_Plugins)以自动化iptables计数器收集。

### 接口级计数器

由于每个容器都有一个虚拟以太网接口，您可能需要直接检查此接口的TX和RX计数器。 你会注意到
每个容器都关联到一个虚拟以太网接口你的主机，用一个名字像`vethKk8Zqi`。 指出输出哪个接口对应于哪个容器，遗憾的是，难。


但现在，最好的方法是检查指标*内容器*。 要完成此操作，您可以从主机运行可执行文件环境在一个容器的网络命名空间内使用** ip-netns魔法**。

命令的确切格式为：

```bash
$ ip netns exec <nsname> <command...>
```

例如:

```bash
$ ip netns exec mycontainer netstat -i
```

`ip netns`查找“mycontainer”容器使用命名空间伪文件。 每个进程属于一个网络命名空间，一个PID命名空间，一个`mnt`命名空间，等等，并且那些命名空间被实现在下`/proc/<pid>/ns/`下。 例如，网络PID 42的命名空间由伪文件实现`/proc/42/ns/net`。

当你运行`ip netns exec mycontainer ...'` 时，它期望`/var/run/netns/ mycontainer`是其中之一那些伪文件。（接受符号链接。）

换句话说，要在a的网络命名空间中执行命令容器，我们需要：

- 找出我们想要调查的容器中的进程的PID;
- 从`/var/run/netns/<somename>`到`/proc/<thepid>/ns/net` 创建一个符号链接
- 执行 `ip netns exec <somename> ....`

请查看[枚举Cgroups](＃enumerating-cgroups)以了解如何查找在你想要的容器中运行的进程的cgroup测量网络使用。 从那里，您可以检查名为`tasks`的伪文件，其中包含的PID对照组（即在容器中）。 选择任何一个。

把所有的东西放在一起，如果容器的“短ID”被保持在环境变量`$ CID`，那么你可以这样做：

```bash
$ TASKS=/sys/fs/cgroup/devices/docker/$CID*/tasks
$ PID=$(head -n 1 $TASKS)
$ mkdir -p /var/run/netns
$ ln -sf /proc/$PID/ns/net /var/run/netns/$CID
$ ip netns exec $CID netstat -i
```

## 高性能指标收集的提示

请注意，每次要更新指标时都要运行新进程（相对）昂贵。 如果你想收集指标高分辨率和/或大量的容器(想1000容器在单个主机上），你不想分叉一个新的进程时间

以下是如何从单个进程收集指标。 你不得不请使用C（或任何允许您执行的语言）编写指标收集器
低级系统调用）。 你需要使用一个特殊的系统调用，`setns（）`，它允许当前进程输入任何
任意命名空间。 然而，它需要一个打开的文件描述符命名空间伪文件（记住：这是伪文件
`/proc/<pid>/ns/ net`）。

但是，有一个catch：你不能保持这个文件描述符打开。如果你这样做，当控制组的最后一个进程退出时，
命名空间不会被销毁，并且其网络资源（如容器的虚拟接口）将永远保持（或直到你关闭文件描述符）。

正确的方法是跟踪每个的第一个PID容器，并且每次重新打开命名空间伪文件。

## 容器退出时收集指标

有时，你不关心实时指标收集，但当一个容器退出，你想知道有多少CPU，内存等被用过。

Docker使这很难，因为它依赖于`lxc-start`，它在退出时仔细清理自己， 它通常更容易定期收集指标（例如，每分钟，与collectd LXC插件），并依赖于它。

但是，如果你仍然想要收集当一个容器停止时资源的统计，
这里是如何做：

对于每个容器，启动收集过程，并将其移动到控制组，通过将其PID写入任务来监视文件的cgroup。 收集过程应定期重读任务文件以检查它是否是控制组的最后一个进程。（如果您还想收集网络统计信息，如中所述
上一节，你也应该把进程移到适当的地方网络命名空间。）

当容器退出时，`lxc-start`会尝试删除控制组。 它将失败，因为控制组是仍在使用中; 但是很好。 你的进程现在应该检测到它只有一个留在组中。 现在是正确的时间收集所有您需要的指标！


最后，你的进程应该回到根控制组，并删除容器控件组。 要删除控制组，只是`rmdir`其目录。 这是反直觉`rmdir`一个目录，因为它仍然包含文件; 但记住这是一个伪文件系统，所以通常的规则不适用。清理完成后，收集过程可以安全退出。
