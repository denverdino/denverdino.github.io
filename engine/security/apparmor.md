---
description: Enabling AppArmor in Docker
keywords: AppArmor, security, docker, documentation
title: AppArmor security profiles for Docker
---

AppArmor (Application Armor) 是用于保护操作系统和应用程序不受安全威胁的Linux安全模块。系统管理员可以为每个程序关联对应的AppArmor profile配置。Docker可以找到并使用配置好的AppArmor策略。

Docker会自动加载容器的AppArmor配置。Docker会在`/etc/apparmor.d/docker`目录下安装一个`docker-default` profile。要注意的是这个文件用于配置容器的安全策略，而非Docker Daemon的安全策略。

现在使用`deb`包安装Docker的时候并不会安装Daemon所对应的AppArmor profile，如果你对Daemon的AppArmor profile感兴趣，可以在Docker Engine的源码仓库里找到[contrib/apparmor](https://github.com/docker/docker/tree/master/contrib/apparmor)

## 理解policies

`docker-default` profile是运行容器是的默认profile，能兼容大多数的应用程序。Profile如下：

```
#include <tunables/global>


profile docker-default flags=(attach_disconnected,mediate_deleted) {

  #include <abstractions/base>


  network,
  capability,
  file,
  umount,

  deny @{PROC}/{*,**^[0-9*],sys/kernel/shm*} wkx,
  deny @{PROC}/sysrq-trigger rwklx,
  deny @{PROC}/mem rwklx,
  deny @{PROC}/kmem rwklx,
  deny @{PROC}/kcore rwklx,

  deny mount,

  deny /sys/[^f]*/** wklx,
  deny /sys/f[^s]*/** wklx,
  deny /sys/fs/[^c]*/** wklx,
  deny /sys/fs/c[^g]*/** wklx,
  deny /sys/fs/cg[^r]*/** wklx,
  deny /sys/firmware/efi/efivars/** rwklx,
  deny /sys/kernel/security/** rwklx,
}
```

当你启动一个容器，只要你没覆盖`security-opt`选项它使用`docker-default`策略，下面的例子明确的指定了默认的策略：

```bash
$ docker run --rm -it --security-opt apparmor=docker-default hello-world
```

## 加载和下载profiles

要为容器加载一个新的profile到Apparmor里：

```bash
$ apparmor_parser -r -W /path/to/your_profile
```
然后，通过`--security-opt`运行自定义的profile：

```bash
$ docker run --rm -it --security-opt apparmor=your_profile hello-world
```

要从AppArmor里卸载profile：

```bash
# 停止AppArmor
$ /etc/init.d/apparmor stop
# 卸载profile
$ apparmor_parser -R /path/to/profile
# 启动AppArmor
$ /etc/init.d/apparmor start
```

### 学习资源

AppArmor中文件名通配的语法和其他文件名通配的实现略有区别，强烈建议你先阅读下面关于AppArmor profile语法的资源：

- [Quick Profile Language](http://wiki.apparmor.net/index.php/QuickProfileLanguage)
- [Globbing Syntax](http://wiki.apparmor.net/index.php/AppArmor_Core_Policy_Reference#AppArmor_globbing_syntax)

## 示例：Nginx profile

在下面的例子里，你创建一个自定义的Nginx AppArmor profile，如下：

```
#include <tunables/global>


profile docker-nginx flags=(attach_disconnected,mediate_deleted) {
  #include <abstractions/base>

  network inet tcp,
  network inet udp,
  network inet icmp,

  deny network raw,

  deny network packet,

  file,
  umount,

  deny /bin/** wl,
  deny /boot/** wl,
  deny /dev/** wl,
  deny /etc/** wl,
  deny /home/** wl,
  deny /lib/** wl,
  deny /lib64/** wl,
  deny /media/** wl,
  deny /mnt/** wl,
  deny /opt/** wl,
  deny /proc/** wl,
  deny /root/** wl,
  deny /sbin/** wl,
  deny /srv/** wl,
  deny /tmp/** wl,
  deny /sys/** wl,
  deny /usr/** wl,

  audit /** w,

  /var/run/nginx.pid w,

  /usr/sbin/nginx ix,

  deny /bin/dash mrwklx,
  deny /bin/sh mrwklx,
  deny /usr/bin/top mrwklx,


  capability chown,
  capability dac_override,
  capability setuid,
  capability setgid,
  capability net_bind_service,

  deny @{PROC}/{*,**^[0-9*],sys/kernel/shm*} wkx,
  deny @{PROC}/sysrq-trigger rwklx,
  deny @{PROC}/mem rwklx,
  deny @{PROC}/kmem rwklx,
  deny @{PROC}/kcore rwklx,
  deny mount,
  deny /sys/[^f]*/** wklx,
  deny /sys/f[^s]*/** wklx,
  deny /sys/fs/[^c]*/** wklx,
  deny /sys/fs/c[^g]*/** wklx,
  deny /sys/fs/cg[^r]*/** wklx,
  deny /sys/firmware/efi/efivars/** rwklx,
  deny /sys/kernel/security/** rwklx,
}
```

1. 保存profile到`/etc/apparmor.d/containers/docker-nginx`文件。 你也可以选择其他文件路径。

2. 加载profile

    ```bash
    $ sudo apparmor_parser -r -W /etc/apparmor.d/containers/docker-nginx
    ```

3. 使用新创建的profile启动容器

    在detached模式中运行Nginx：

    ```bash
    $ docker run --security-opt "apparmor=docker-nginx" \
        -p 80:80 -d --name apparmor-nginx nginx
    ```

4. 在容器里执行Exec

    ```bash
    $ docker exec -it apparmor-nginx bash
    ```

5. 通过一些操作测试新创建的profile

    ```bash
    root@6da5a2a930b9:~# ping 8.8.8.8
    ping: Lacking privilege for raw socket.

    root@6da5a2a930b9:/# top
    bash: /usr/bin/top: Permission denied

    root@6da5a2a930b9:~# touch ~/thing
    touch: cannot touch 'thing': Permission denied

    root@6da5a2a930b9:/# sh
    bash: /bin/sh: Permission denied

    root@6da5a2a930b9:/# dash
    bash: /bin/dash: Permission denied
    ```

恭喜！你已经通过自定义的apparmor profile让部署的容器更加安全。

## 调试arrarmor

你可以通过`dmesg`调试问题，通过`aa-status`校验已加载的profiles。

### 使用dmseg

使用AppArmor的时候你可能会遇到一些问题，这里有一些帮你调试问题的小贴士。

AppArmor向`dmesg`发送详尽的信息，通常一行AppArmor日志类似下面这样：

```
[ 5442.864673] audit: type=1400 audit(1453830992.845:37): apparmor="ALLOWED" operation="open" profile="/usr/bin/docker" name="/home/jessie/docker/man/man1/docker-attach.1" pid=10923 comm="docker" requested_mask="r" denied_mask="r" fsuid=1000 ouid=0
```

在上面的例子里，你可以看到`profile=/usr/bin/docker`，它的意思是`docker-engine` profile已经加载了。

> **注意：** Ubuntu 14.04以上的版本可以正常工作。Ubuntu Trusty上使用`docker exec`可能会有问题。

再看看另外一行日志：

```
[ 3256.689120] type=1400 audit(1405454041.341:73): apparmor="DENIED" operation="ptrace" profile="docker-default" pid=17651 comm="docker" requested_mask="receive" denied_mask="receive"
```

这次profile是`docker exec`，就是启动容器是默认使用的profile（除非你使用`privileged`模式）。这行日志显示在容器里`ptrace`操作被拒绝了，符合预期。

### 使用 aa-status

如果你想检查有哪些profiles被加载了，可以使用`aa-status`，输出如下

```bash
$ sudo aa-status
apparmor module is loaded.
14 profiles are loaded.
1 profiles are in enforce mode.
   docker-default
13 profiles are in complain mode.
   /usr/bin/docker
   /usr/bin/docker///bin/cat
   /usr/bin/docker///bin/ps
   /usr/bin/docker///sbin/apparmor_parser
   /usr/bin/docker///sbin/auplink
   /usr/bin/docker///sbin/blkid
   /usr/bin/docker///sbin/iptables
   /usr/bin/docker///sbin/mke2fs
   /usr/bin/docker///sbin/modprobe
   /usr/bin/docker///sbin/tune2fs
   /usr/bin/docker///sbin/xtables-multi
   /usr/bin/docker///sbin/zfs
   /usr/bin/docker///usr/bin/xz
38 processes have profiles defined.
37 processes are in enforce mode.
   docker-default (6044)
   ...
   docker-default (31899)
1 processes are in complain mode.
   /usr/bin/docker (29756)
0 processes are unconfined but have a profile defined.
```

上面的输出说明`docker-default`运行在`enforce`模式，意味着AppArmor已经激活，正在阻止所有在`docker-default` profile之外的操作，并且审计到`dmesg`中。

上面的输出同样显示了 `/usr/bin/docker` profile运行在`complain`模式下，意味着AppArmor**仅仅**记录profile之外的操作，而非阻止。

## 向Docker's AppArmor贡献代码

高级用户和包管理员可以在Docker源码仓库下的[contrib/apparmor](https://github.com/docker/docker/tree/master/contrib/apparmor)里找到`/usr/bin/docker`的profile

容器默认的`docker-default` profile位于[profiles/apparmor](https://github.com/docker/docker/tree/master/profiles/apparmor).
