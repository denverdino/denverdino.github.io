---
description: Describes how to use the fluentd logging driver.
keywords: Journald, docker, logging, driver
redirect_from:
- /engine/reference/logging/journald/
title: Journald logging driver
---

`journald`日志驱动程序将容器日志发送到[systemd journald](http://www.freedesktop.org/software/systemd/man/systemd-journald.service.html)。
日志消息可以使用`journalctl`命令、journal API、或使用`docker logs`命令获取。

除了日志消息本身的文本，`journald`日志驱动程序在每个消息的日志中存储以下元数据：

|字段|说明|
---------------------- | ------------- |
| `CONTAINER_ID` |容器ID截断为12个字符。 |
| `CONTAINER_ID_FULL` |完整的64个字符的容器ID。 |
| `CONTAINER_NAME` |启动时的容器名称。如果使用`docker rename`重命名容器，新名称不会反映在日记中。 |
| `CONTAINER_TAG` |容器标签（[日志标签文档](log_tags.md)）。 |

## 用法

您可以通过设置`--log-driver'参数来给Docker守护进程配置默认日志驱动程序：

    dockerd --log-driver=journald

您可以在`docker run`时使用`--log-driver`选项为特定容器设置日志驱动程序：

    docker run --log-driver=journald ...


## 选项

用户可以使用`--log-opt NAME=VALUE`参数来指定journald日志驱动程序的其他选项。

### 标签

指定模板以在journald日志中设置“CONTAINER_TAG”值。自定义日志标记格式请参考[日志标签文档](log_tags.md)。

### 标签和env

`labels`和`env`选项接受使用逗号分隔的键列表。如果`label`和`env`之间有冲突，`env`的值优先。两个选项都会在日志中添加元数据。

## 关于容器名的注释

“CONTAINER_NAME”字段中记录的值是容器启动时的名称。如果使用`docker rename`重命名容器，新的名称
将不会反映在日记中。日记将继续使用原名称。

## 使用journalctl检索日志消息

您可以使用`journalctl`命令检索日志消息。

您可以使用过滤器表达式来检索特定容器的日志。例如，要从所有日志中检索出特定名字容器的日志：

    # journalctl CONTAINER_NAME=webserver

您可以使用其他过滤器来进一步限定日志检索。例如，只查看上次系统重启之后的日志：

    # journalctl -b CONTAINER_NAME=webserver

或者检索JSON格式的带有完整元数据的日志：

    # journalctl -o json CONTAINER_NAME=webserver

## 使用journal API检索日志消息

这个例子使用`systemd` Python模块来检索容器日志：

    import systemd.journal

    reader = systemd.journal.Reader()
    reader.add_match('CONTAINER_NAME=web')

    for msg in reader:
      print '{CONTAINER_ID_FULL}: {MESSAGE}'.format(**msg)

