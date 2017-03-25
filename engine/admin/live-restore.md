---
description: How to keep containers running when the daemon isn't available.
keywords: docker, upgrade, daemon, dockerd, live-restore, daemonless container
title: 守护进程停止时保持容器运行
---

默认设置下，当 Docker 守护进程退出时，它将关闭所有正在运行的容器。从 Docker Engine
1.12 开始，你可以通过配置来使得容器在守护进程不可用的情况下仍然保持运行。live restore
选项可以帮助我们减少因守护进程崩溃、维护或者升级而造成的容器停机时间。

> **注意**：Live restore 不支持 Windows 容器，但可以支持在 Windows 平台上运行的 Linux
 容器

## 启用 live restore 选项

有两种方法启用 live restore：

* 如果 Docker 守护进程已经在运行了，并且你又不想终止它，那么可以将配置添加到守护进程的
配置文件中。例如，在 Linux 系统中默认的配置文件是 `/etc/docker/daemon.json`.

用你最喜欢的编辑器在 `daemon.json` 文件中打开 `live-restore` 选项

```bash
{
"live-restore": true
}
```

你需要向守护进程发送一个 `SIGHUP` 信号来通知守护进程重新加载配置文件。更多关于使用
 config.json 来配置 Docker 守护进程的信息可查看[守护进程配置文件](../reference/commandline/dockerd.md#daemon-configuration-file).

* 在启动 Docker 守护进程时，向命令传递 `--live-restore` 参数

    ```bash
    $ sudo dockerd --live-restore
    ```

## 升级期间的 live restore

live restore 选项支持守护进程从一个小版本升级到下一个，例如 Docker Engine 从
1.12.1 升级到 1.13.2.

如果你在升级时跳过了一些版本号，那么守护进程可能无法重新连接到此前运行的容器。如果守护
进程无法重建连接，它将无视这些运行中的容器，你必须手动管理它们。守护进程不会关闭这些失去
连接的容器。

## 重启时的 live restore

live restore 选项只能支持守护进程重启前后配置选项相同的情况。例如，如果守护进程重启后的
bridge IP 或 graphdriver 与之前不一样，那么 live restore 可能无法工作。

## live restore 对正在运行的容器的影响

守护进程长时间不工作将影响到运行中的容器。容器进程会写 FIFO 的日志给守护进程，如果守护
进程无法及时处理掉这些输出，缓冲区将被填满，并且接下来向日志的写入会被阻塞住。直到有更多的缓冲空间可用前，写满的日志会阻塞住容器进程。默认的缓冲大小一般是 64K.

你必须重新启动 Docker 来刷掉缓冲。

你可以通过更改 `/proc/sys/fs/pipe-max-size` 来修改内核的缓冲区大小。

## live restore 和 swarm 模式

live restore 选项不兼容 Docker Engine 的 swarm 模式。当 Docker Engine 处于 swarm 模式时，
orchestration 功能会管理任务并且使容器根据服务规范来保持运行。
