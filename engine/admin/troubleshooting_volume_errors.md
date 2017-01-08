---
description: Troubleshooting volume errors
keywords:
- cadvisor, troubleshooting, volumes, bind-mounts
title: Troubleshoot volume errors
---

# 排除卷错误

本主题讨论在使用或绑定Docker卷时可能出现的错误。

## `Error: Unable to remove filesystem`

一些基于容器的实用程序，如[Google cAdvisor](https://github.com/google/cadvisor)，会把Docker系统目录如`/var/lib/docker/`挂载到容器中。比如，“cadvisor”的文档指示您以如下方式运行`caditor`容器：

```bash
$ sudo docker run \
  --volume=/:/rootfs:ro \
  --volume=/var/run:/var/run:rw \
  --volume=/sys:/sys:ro \
  --volume=/var/lib/docker/:/var/lib/docker:ro \
  --publish=8080:8080 \
  --detach=true \
  --name=cadvisor \
  google/cadvisor:latest
```

当bind-mount`/var/lib/docker/`时，相当于把所有其他正在运行的容器的资源挂载到了本容器中。当您尝试删除任何这些容器时都可能会失败，并显示以下错误：

```none
Error: Unable to remove filesystem for
74bef250361c7817bee19349c93139621b272bc8f654ae112dd4eb9652af9515:
remove /var/lib/docker/containers/74bef250361c7817bee19349c93139621b272bc8f654ae112dd4eb9652af9515/shm:
Device or resource busy
```

当挂载了`/var/lib/docker/`的容器对`/var/lib/docker/`下的文件句柄使用了`statfs`或`fstatfs`但又没有关闭时，会发生该问题。 

通常，我们不建议用这种方式挂载`/var/lib/docker/`。然而，`cAdvisor`需要这个bind-mount来实现核心功能。

如果您不确定哪个进程导致了错误中提到的路径忙和无法删除，你可以使用`lsof`命令找到该进程。例如，对于上面的错误：

```bash
$ sudo lsof /var/lib/docker/containers/74bef250361c7817bee19349c93139621b272bc8f654ae112dd4eb9652af9515/shm
```

要解决此问题，请停止挂载了`/var/lib/docker` 的容器，然后再次尝试删除其他容器。

