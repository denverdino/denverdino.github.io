---
redirect_from:
  - "/engine/articles/systemd/"
title: "Limit a container's resources"
description: "Limiting the system resources a container can use"
keywords: "docker, daemon, configuration"
---

默认情况下，一个容器是没有资源限制的，它可以使用宿主机上 kernel 调度器提供的所有资源。Docker 提供多种方式去控制容器的内存， CPU， block IO 使用量， 在`docker run`中就可以使用运行时参数去控制它们。这一节会提供关于如何配置这些限制的详细细节,并且会给这些设置提供一些建议。

## 内存

Docker 可以实施强内存限制，这样容器就只能使用声明的内存。同时Docker 还提供软限制，这样可以让容器使用尽可能多的内存，除非遇到一些情况，比如当kernel发现内存不够用或者宿主机有资源冲突的时候。这些可选项在单独使用和混合使用的时候会有不同的影响。

大部分选项都是正整数，前缀 `b`, `k`,
`m`, `g`, 表明 bytes, kilobytes, megabytes, 或者 gigabytes.


| 选项                | 描述                 |
|-----------------------|-----------------------------|
| `-m` 或者 `--memory=` | 容器可以使用的最大内存。如果您设定这个选项，最小值为 `4m`|
| `--memory-swap`*    | 容器可以交换到磁盘的内存量 详情查看[`--memory-swap`](resource_constraints.md#memory-swap-details)|
| `--memory-swappiness` | 默认情况下,主机内核可以换出的容器使用的匿名页面百分比。你可以设置 `--memory-swappiness` 0-100 去改变这个百分比。查看详情 [`--memory-swappiness`](resource_constraints.md#memory-swappiness-details)|
| `--memory-reservation` | 允许您指定一个软限制小于`--memory`， 他会当Docker 检测到主机上发生资源争抢或者内存不够用时触发。如果您设置`--memory-reservation`， 为了优先考虑`--memory`, 它必须小于`--memory`。 因为它是软限制， 它不保证容器不会超出这个限制。|
| `--kernel-memory` | The maximum amount of kernel memory the container can use. The minimum allowed value is `4m`. Because kernel memory cannot be swapped out, a container which is starved of kernel memory may block host machine resources, which can have side effects on the host machine and on other containers. 容器内核内存能够使用的最大量。 最小值为 `4m`。因为内核内存不能被换出,一个容器的内核内存匮乏可能会阻碍主机资源,这可能对宿主机和其他容器有负面影响。 详情 [`--kernel-memory`](resource_constraints.md#kernel-memory-details)。 |
| `--oom-kill-disable` | By default, if an out-of-memory (OOM) error occurs, the kernel kills processes in a container. To change this behavior, use the `--oom-kill-disable` option. Only disable the OOM killer on containers where you have also set the `-m/--memory` option. If the `-m` flag is not set, the host can run out of memory and the kernel may need to kill the host system's processes to free memory. 默认情况下,如果出现内存不足错误(OOM),内核 就会杀死容器进程。改变这种行为， 使用`--oom-kill-disable`选项。只有设置`-m/--memory` 选项的时候才会去关闭 OOM kill 。如果没有设置`- m`标志, 主机可以耗尽内存，内核可能需要杀死主机系统的进程释放内存 |

关于cgroups 和内存的更多信息，请阅读 [内存资源控制](https://www.kernel.org/doc/Documentation/cgroup-v1/memory.txt).

### `--memory-swap` 详情

- 如果不设置, 同时 `--memory` 设置了, 如果容器有交换内存设置，容器可以使用两倍的 `--memory` 设置。例如, 如果 `--memory="300m"` 然后 `--memory-swap` 没有设置, 容器可以使用300m 内存和600m 交换分区。
 
- 如果设置为正整数, 同时 `--memory` 和 `--memory-swap`都设置了, `--memory-swap` 代表内存和可交换区的总量, `--memory` 控制非交换内存使用量。 因此，如果 `--memory="300m"` 同时 `--memory-swap="1g"`, 容器可以使用300m 内存和 700m (1g - 300m) 交换分区。

- 如果设置为 `-1` (默认), 容器允许使用无限交换内存。

### `--memory-swappiness` 详情

- 0 关闭匿名交换页
- 100 设置所有匿名页为可交换的
- 默认，如果您没有设置 `--memory-swappiness`， 这个值会继承自宿主机。

### `--kernel-memory` 详情

内核内存限制表达为总体分配给一个容器的内存。考虑以下场景:

- **无限内存，无限内核内存**: 默认行为
- **无限内存，有限内核内存**: 当所有容器所需的内存大于实际宿主机内存时，这是合适的。您可以配置内核内存不要超过主机,这时需要更多内存的容器就需要等待。
- **有限内存, 无限内核内存**: 整个内存是有限的,但内核内存不是
- **有限内存，有限内核内存**:  限制用户和内核内存有利于调试与内存相关的问题。如果容器是使用一个意想不到的类型的内存,这将耗尽内存,而不影响其他容器或主机。有了这个设置,如果内核内存限制低于用户内存限制,内核内存耗尽将导致容器 OOM。如果内核内存限制高于用户内存限制,内核限制不会导致容器OOM。

When you turn on any kernel memory limits, the host machine tracks "high water
mark" statistics on a per-process basis, so you can track which processes (in
this case, containers) are using excess memory. This can be seen per process
by viewing `/proc/<PID>/status` on the host machine.
当你打开任何内核内存限制,主机就会在统计每个进程的基础上追踪“高水标”进程,所以你可以追踪哪些进程(在本例中,容器)使用多余的内存。这可以通过在宿主机上查看 `/proc/<PID>/status`。

## CPU

默认情况下，每个容器访问主机的CPU周期是无限制的。 您可以设置各种约束以限制给定容器访问主机的CPU周期。

| 选项                | 描述                 |
|-----------------------|-----------------------------|
| `--cpu-shares` | 将此标志设置为大于或小于默认值1024的值，以增加或减少容器的权重，并使其访问主机CPU周期以更大或更小的比例。 这只有在CPU周期受到限制时才会强制执行。 当大量的CPU周期可用时，所有容器使用所需的CPU数量。 这样，这是一个软限制。 `--cpu-share`不会阻止容器在swarm mode 模式下调度。 它为可用的CPU周期优先化容器CPU资源。 它不保证或保留任何特定的CPU访问。|
| `--cpu-period` | 容器上一个逻辑CPU的调度周期。 `--cpu-period`默认为时间值100000（100 ms）。 |
| `--cpu-quota` |  在由`--cpu-period`设置的时间段内容器可以调度的最大时间量。|
| `--cpuset-cpus` | 使用此选项将容器固定到一个或多个CPU核心，并以逗号分隔。 |

### `--cpu-period` 和 `--cpu-qota` 例子

如果您有1个vCPU系统，并且容器使用`--cpu-period = 100000`和`--cpu-quota = 50000`运行，则容器最多可以占用1个CPU的50％。

```bash
$ docker run -ti --cpu-period=10000 --cpu-quota=50000 busybox
```

如果您有一个4 vCPU系统，您的容器使用`--cpu-period = 100000`和`--cpu-quota = 200000`，您的容器可以使用最多2个逻辑CPU（`--cpu-period`的200％）。

```bash
$ docker run -ti --cpu-period=100000 --cpu-quota=200000
```

### `--cpuset-cpus` 例子

要给容器准确访问4个CPU，请发出如下命令：

```bash
$ docker run -ti --cpuset-cpus=4 busybox
```

## Block IO (blkio)

有两个选项可用于调整给定容器直接对块IO设备的访问。 您还可以按照每秒的字节数或每秒的IO操作来指定带宽限制。

| 选项                | 描述                 |
|-----------------------|-----------------------------|
| `blkio-weight` | 默认权重为500.要提高或降低给定容器使用的blkio的比例，请将`--blkio-weight`标志设置为介于10和1000之间的值。此设置会平等影响所有块IO设备。|
| `blkio-weight-device` |  它与`--blkio-weight`相同，但您可以使用语法`--blkio-weight-device =“DEVICE_NAME：WEIGHT”`为每个设备设置权重。DEVICE_NAME：WEIGHT是一个字符串，包含冒号分隔的设备名称和权重 。|
| `--device-read-bps` 和 `--device-write-bps` | 根据大小限制设备的读取或写入速率，使用kb，mb或gb后缀。|
| `--device-read-iops` 或者 `--device-write-iops` | Limits the read or write rate to or from a device by IO operations per second. 通过每秒的IO操作来限制设备的读取或写入速率。|


### Block IO 权重例子

>**注意**: `--blkio-weight`标志只影响直接IO，对缓冲IO没有影响。

如果指定`--blkio-weight`和`-blkio-weight-device`，Docker使用`--blkio-weight`作为默认权重，并使用`--blkio-weight-device`重写命名设备上的默认值。

要将`/dev/sda` 的容器的设备权重设置为200，而不指定默认`blkio-weight`：

```bash
$ docker run -it \
  --blkio-weight-device "/dev/sda:200" \
  ubuntu
```

### 块带宽限制示例

此示例将`ubuntu`容器的`/dev/sda`最大写入速度限制为为1mbps：

```bash
$ docker run -it --device-write-bps /dev/sda:1mb ubuntu
```

此示例将`ubuntu`容器限制为从`/dev/sda`到每秒1000次IO操作的最大读取速率：

```bash
$ docker run -ti --device-read-iops /dev/sda:1000 ubuntu
```
