---
description: Enabling seccomp in Docker
keywords: seccomp, security, docker, documentation
title: Seccomp security profiles for Docker
---

安全计算模型(Seccomp)是Linux内核的一个特性。你可以用它限制容器内可以执行操作。系统调用`seccomp()`能操作调用进程的seccomp状态，你可以用这个特性限制你应用程序的行为。


仅当Docker构建的时候启用了seccomp，并且内核配置里开启了`CONFIG_SECCOMP`，要检查内核是否支持seccomp：

```bash
$ cat /boot/config-`uname -r` | grep CONFIG_SECCOMP=
CONFIG_SECCOMP=y
```

> **注意**: seccomp profiles需要seccomp 2.2.1 而且操作系统要求Debian 9 "Stretch", Ubuntu 15.10 "Wily", Fedora 22, CentOS 7, Oracle Linux 7. 要在Ubuntu 14.04, Debian Wheezy或者Debian Jessie上使用此特性，你必须下载[最新版本静态编译的Linux Docker](../installation/binaries.md)。其他发行版还**不**支持这个特性。

## 传给容器一个profile

默认的seccomp profile广泛适用于多数容器，禁用了大约300多个系统调用中的44个，对大多数应用都够用了。默认的Docker profile具有如下形式的JSON结构：

```json
{
	"defaultAction": "SCMP_ACT_ERRNO",
	"architectures": [
		"SCMP_ARCH_X86_64",
		"SCMP_ARCH_X86",
		"SCMP_ARCH_X32"
	],
	"syscalls": [
		{
			"name": "accept",
			"action": "SCMP_ACT_ALLOW",
			"args": []
		},
		{
			"name": "accept4",
			"action": "SCMP_ACT_ALLOW",
			"args": []
		},
		...
	]
}
```

如果你在启动容器时没有指定`security-opt`选项，就会使用默认的profile。下面的例子明确指定使用自定义的profile：

```
$ docker run --rm -it --security-opt seccomp=/path/to/seccomp/profile.json hello-world
```

### 默认profile屏蔽的重要系统调用

Docker默认的seccomp profile提供了一个允许的系统调用的白名单。下面的表格列出了不包含在白名单里的重要系统调用（不是全部）。表格里包含每个系统调用被禁止的原因。


| Syscall             | Description                                                                                                                           |
|---------------------|---------------------------------------------------------------------------------------------------------------------------------------|
| `acct`              | 能让容器禁用自身的资源限制和进程计数。CAP_SYS_PACCT也会控制这个功能 |
| `add_key`           | 阻止容器使用内核keyring。后者是内核唯一的，没有namespace化                                   |
| `adjtimex`          | 类似`clock_settime` and `settimeofday`, 时间/日期namespace化                                  |
| `bpf`               | 禁止加载潜在持久化bpf程序到内核里，同时要禁止`CAP_SYS_ADMIN`.              |
| `clock_adjtime`     | 时间/日期没有按namespace隔离.                                                                                 |
| `clock_settime`     | 时间/日期没有按namespace隔离.                                                                                 |
| `clone`             | 禁止克隆一个新的namespace。 `CAP_SYS_ADMIN`也会控制这个功能        |
| `create_module`     | 禁止操作内核模块                                                           |
| `delete_module`     | 禁止操作内核模块。 同样由`CAP_SYS_MODULE`控制                           |
| `finit_module`      | 禁止操作内核模块。 同样由`CAP_SYS_MODULE`控制               |
| `get_kernel_syms`   | 禁止获取内核和模块暴露出的符号                                                        |
| `get_mempolicy`     | 禁止修改内核内存好NUMA设置。`CAP_SYS_NIC`亦然                      |
| `init_module`       | 禁止操作内核模块。 同样由`CAP_SYS_MODULE`控制                          |
| `ioperm`            | 阻止容器修改内核I/O 权限等级。同样由 `CAP_SYS_RAWIO`控制           |
| `iopl`              | 阻止容器修改内核I/O 权限等级。同样由 `CAP_SYS_RAWIO`控制           |
| `kcmp`              | 限制进程自省capabilities, 也可以通过DROP `CAP_PTRACE`限制。                         |
| `kexec_file_load`   | 类似`kexec_load`，参数不一样                   |
| `kexec_load`        | 禁止加载新内核                                                               |
| `keyctl`            | 阻止容器使用内核keyring。后者是内核唯一的，没有namespace化   |
| `lookup_dcookie`    | Tracing/profiling系统调用, 这些操作可能导致泄露大量的宿主机信息。
|
| `mbind`             | 阻止能修改内核内存和NUMA设置的系统调用，也可以通过drop `CAP_SYS_NICE`                     |
| `mount`             | 禁止容器挂载存储, 同样有`CAP_SYS_ADMIN`控制                                                            |
| `move_pages`        | 阻止能修改内核内存和NUMA设置的系统调用                                                       |
| `name_to_handle_at` | 类似`open_by_handle_at`                                   |
| `nfsservctl`        | 阻止和内核里的nfs后台进程交互。                                                                |
| `open_by_handle_at` | 导致内核逃逸. 也可以通过`CAP_DAC_READ_SEARCH`.                                     |
| `perf_event_open`   | Tracing/profiling系统调用, 这些操作可能导致泄露大量的宿主机信息。                                |
| `personality`       | 阻止容器启用BSD仿真，不是特别危险，但是没严格测试过，可能存在内核缺陷 |
| `pivot_root`        | 禁止`pivot_root`                                                      |
| `process_vm_readv`  | 限制进程内省的能力, 也可以通过Drop `CAP_PTRACE`                      |
| `process_vm_writev` | 限制进程内省的能力, 也可以通过Drop `CAP_PTRACE`                       |
| `ptrace`            | Tracing/profiling系统调用, 这些操作可能导致泄露大量的宿主机信息。 也可以通过Drop `CAP_PTRACE`. |
| `query_module`      | 禁止操作内核模块                                                            |
| `quotactl`          | 限制配额相关的系统调用（允许容器禁用自身的资源限制和进程计数），也可以通过Drop `CAP_SYS_ADMIN`. |
| `reboot`            | 禁止容器重启机器。也可以通过Drop `CAP_SYS_BOOT`.                                           |
| `request_key`       | 阻止容器使用内核keyring。后者是内核唯一的，没有namespace化                                    |
| `set_mempolicy`     | 阻止能修改内核内存和NUMA设置的系统调用，也可以通过drop `CAP_SYS_NICE`                   |
| `setns`             | 阻止给线程关联namespace，也可以通过Drop `CAP_SYS_ADMIN`.                                    |
| `settimeofday`      | 时间/日期没有按namespace隔离                                               |
| `stime`             | 时间/日期没有按namespace隔离                                              |
| `swapon`            | 禁止启动或者停止文件以及设备的swap配置                                      |
| `swapoff`           |  禁止启动或者停止文件以及设备的swap配置                            |
| `sysfs`             | 废弃                                                                                             |
| `_sysctl`           | 废弃，用/proc/sys替换
|
| `umount`            | 特权操作，也可以通过Drop `CAP_SYS_ADMIN`.                                              |
| `umount2`           | 特权操作                                                                             |
| `unshare`           | 禁止为进程创建新的namespace
|
| `uselib`            | 共享库相关的系统调用，很久不用了                                  |
| `userfaultfd`       | 用户态页错误处理，进程迁移中广泛适用                                          |
| `ustat`             |废弃                                                                                             |
| `vm86`              | 内核x86实模式虚拟机，也可以通过Drop `CAP_SYS_ADMIN`.                                       |
| `vm86old`           | 内核x86实模式虚拟机，也可以通过Drop `CAP_SYS_ADMIN`
   |

## 不使用默认的seccomp profile启动容器

要启动容器并且不适用默认的seccomp profile，你可以在启动容器的时候传入`unconfined`参数

```
$ docker run --rm -it --security-opt seccomp=unconfined debian:jessie \
    unshare --map-root-user --user sh -c whoami
```
