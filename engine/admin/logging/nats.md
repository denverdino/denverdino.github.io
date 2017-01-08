---
description: Describes how to use NATS for publishing log entries
keywords: NATS, nats.io, messaging, docker, logging, driver
title: NATS logging driver
---

Docker日志驱动程序，用于将日志作为以JSON格式发布到NATS的方式。

## 使用

您可以通过传递`--log-driver`来配置默认日志驱动程序选项到Docker守护进程：

```bash
$ dockerd --log-driver=nats
```

您可以通过使用为特定容器设置日志驱动程序`--log-driver`选项`docker run`：

```bash
$ docker run --log-driver=nats ...
```

这个日志驱动程序没有实现一个读取器，所以它与`docker logs`不兼容。

## nats 选项

您可以使用`--log-opt NAME = VALUE`标志来自定义日志记录驱动程序为NATS：

| 选项                      | 必须 | 描述                                                                                                                                 |
|-----------------------------|----------|---------------------------------------------------------------------------------------------------------------------------------------------|
| `labels`                    | 可选 | 以逗号分隔的标签键列表，如果为容器指定这些标签，则应包含在消息中。                  |
| `env`                       | 可选 | 以逗号分隔的环境变量的键列表，如果为容器指定这些变量，则应包含在消息中。|
| `tag`                       | 可选 | 指定消息的标记。 有关定制日志标记格式的信息，请参阅[log tag选项文档](log_tags.md)。                      |
| `nats-servers`              | 可选 |NATS集群节点之间用逗号分隔。 例如 `nats：//127.0.0.1：4222，nats：//127.0.0.1：4223`。 默认为`localhost：4222`      |            
| `nats-max-reconnect`        | 可选 | 在放弃之前，驱动程序尝试连接的最大尝试次数。 默认为无限（`-1`）                                       |
| `nats-subject`              | 可选 | 将发布日志的具体主题。 默认为使用`tag`（如果未指定）                                                 |
| `nats-user`                 | 可选 |在需要验证的情况下指定用户                                                            |
| `nats-pass`                 | 可选 | 在需要验证的情况下指定密码                                                                       |
| `nats-token`                 | 可选 | 在需要验证的情况下指定令牌                                                          |
| `nats-tls-ca-cert`          | 可选 |指定由CA签名的信任证书的绝对路径                                                            |
| `nats-tls-cert`             | 可选 | 指定TLS证书文件的绝对路径                                                              |
| `nats-tls-key`              | 可选 |指定TLS密钥文件的绝对路径                                                             |
| `nats-tls-skip-verify`      | 可选 | 指定是否通过将验证设置为`true`来跳过验证                                                           |

下面是驱动程序用于将日志发送到节点的示例用法， NATS集群到`docker.logs`主题：

```bash
$ docker run --log-driver=nats \
             --log-opt nats-subject=docker.logs \
             --log-opt nats-servers=nats://nats-node-1:4222,nats://nats-node-2:4222,nats://nats-node-3:4222 \
             your/application
```


默认情况下，标记用作NATS的主题，因此它必须是有效的主题，如果主题未指定：

```bash
{% raw %}
$ docker run --log-driver nats \
             --log-opt tag="docker.{{.ID}}.{{.ImageName}}"
             your/application
{% endraw %}
```

使用TLS的安全连接到NATS可以通过设置`tls：//'方案来定制在URI和绝对路径中的证书和密钥文件：

```bash
docker run --log-driver nats \
           --log-opt nats-tls-key=/srv/configs/certs/client-key.pem \
           --log-opt nats-tls-cert=/srv/configs/certs/client-cert.pem \
           --log-opt nats-tls-ca-cert=/srv/configs/certs/ca.pem \
           --log-opt nats-servers="tls://127.0.0.1:4223,tls://127.0.0.1:4222" \
           your/application
```

默认情况下启用跳过验证，为了停用，我们可以指定`nats-tls-skip-verify`：

```bash
  docker run --log-driver nats \
             --log-opt nats-tls-skip-verify \
             --log-opt nats-servers="tls://127.0.0.1:4223,tls://127.0.0.1:4222" \
             your/application
```
