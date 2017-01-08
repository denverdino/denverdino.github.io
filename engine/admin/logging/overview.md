---
description: Configure logging driver.
keywords: docker, logging, driver, Fluentd
redirect_from:
title: Configure logging drivers
---

Docker支持多个日志记录机制。在启动“dockerd”守护程序时使用`--log-driver=VALUE`参数设置默认的日志记录
驱动程序。您可以使用`--log-opt NAME=VALUE`参数来指定驱动程序的其他选项。

容器可以有不同于Docker守护进程的日志驱动程序。在`docker run`时使用`--log-driver=VALUE`参数来配置容器的日志驱动程序。

如果没有设置`-log-driver`选项，容器
使用守护程序的默认日志驱动程序，默认为`json-file`。

如果要检查Docker守护程序的默认日志驱动程序，可以在`docker info`命令的输出在中搜索“Logging Driver”：

```bash
$ docker info |grep Logging

Logging Driver: json-file
```

支持以下选项：

| 驱动 | 说明 |
| ------------- | ---------------------------------------------------------------------------------------- |
| `none` | 禁用容器的任何日志记录。`docker logs`不能用于此驱动程序。 |
| `json-file` | Docker的默认日志驱动程序。将JSON格式的日志写入文件。 |
| `syslog` | Docker的Syslog日志驱动程序。将日志消息写入syslog。 |
| `journald` | Docker的Journald日志记录驱动程序。将日志消息写入`journald`。 |
| `gelf` | Docker的Graylog扩展日志格式（GELF）日志驱动程序。将日志消息写入GELF端点，如Graylog或Logstash。 |
| `fluentd` | Docker的Fluentd日志驱动程序。将日志消息写入`fluentd`（正向输入）。 |
| `awslogs` | Docker的Amazon CloudWatch Logs日志驱动程序。将日志消息写入Amazon CloudWatch Logs。 |
| `splunk` | Docker的Splunk日志驱动程序。使用HTTP事件收集器将日志消息写入`splunk'。 |
| `etwlogs` | Windows Docker的ETW日志驱动程序。将日志消息作为ETW事件写入。 |
| `gcplogs` | Docker的Google Cloud Logging驱动程序。将日志消息写入Google Cloud Logging。 |
| `nats` | Docker的NATS日志驱动程序。将日志条目发布到NATS服务器。 |

`docker logs`命令仅适用于`json-file`和`journald`日志驱动程序。

`labels`和`env`选项为日志驱动程序增加额外的属性。每个选项都是逗号分隔的键列表。如果在`label`和`env`的键之间有冲突，`env`的值优先。

在启动Docker守护程序时指定属性。例如，以下命令使用`json-file`驱动程序启动守护程序并包括其他属性。

```bash
$ dockerd --log-driver=json-file \
          --log-opt labels=foo \
          --log-opt env=foo,fizz
```

此命令运行具有特定`labels`或`env`的容器。

```bash
$ docker run -dit --label foo=bar -e fizz=buzz alpine sh
```

这会根据驱动程序向日志中添加其他字段。如果你使用`json-file`，它可能添加的属性如下：

```json
"attrs":{"fizz":"buzz","foo":"bar"}
```

## json-file选项

`json-file'日志驱动程序支持以下选项：

```none
--log-opt max-size=[0-9]+[kmg]
--log-opt max-file=[0-9]+
--log-opt labels=label1,label2
--log-opt env=env1,env2
```

达到`max-size`的日志会轮转。您可以设置千字节（k），兆字节（m）或千兆字节（g）。例如`--log-opt max-size=50m`。如果`max-size`未设置，则日志不会翻转。

`max-file`指定保留多少个日志文件。例如`--log-opt max-file=100`。如果没有设置`max-size`，那么`max-file`不生效。

如果设置了`max-size`和`max-file`，`docker logs`只返回最新日志文件中的日志。


## syslog选项

`syslog'日志驱动程序支持以下日志记录选项：

```none
--log-opt syslog-address=[tcp|udp|tcp+tls]://host:port
--log-opt syslog-address=unix://path
--log-opt syslog-address=unixgram://path
--log-opt syslog-facility=daemon
--log-opt syslog-tls-ca-cert=/etc/ca-certificates/custom/ca.pem
--log-opt syslog-tls-cert=/etc/ca-certificates/custom/cert.pem
--log-opt syslog-tls-key=/etc/ca-certificates/custom/key.pem
--log-opt syslog-tls-skip-verify=true
--log-opt tag="mailer"
--log-opt syslog-format=[rfc5424|rfc5424micro|rfc3164]
--log-opt env=ENV1,ENV2,ENV3
--log-opt labels=label1,label2,label3
```

`syslog-address`指定驱动程序要连接的远程syslog服务器地址。如果没有指定，它默认为本地系统的unix socket。 如果指定了`tcp`或`udp`但`port`未指定，默认为`514`。

以下示例显示如何让`syslog`驱动程序连接到192.168.0.42服务器，端口`123`

```bash
$ docker run --log-driver=syslog --log-opt syslog-address=tcp://192.168.0.42:123
```

`syslog-facility`选项配置syslog工具。 默认情况下，系统使用`daemon`值。 要覆盖此行为，您可以提供0到23的整数或任何以下名字：

* `kern`
* `user`
* `mail`
* `daemon`
* `auth`
* `syslog`
* `lpr`
* `news`
* `uucp`
* `cron`
* `authpriv`
* `ftp`
* `local0`
* `local1`
* `local2`
* `local3`
* `local4`
* `local5`
* `local6`
* `local7`

`syslog-tls-ca-cert`指定由CA签名的信任证书的绝对路径。 如果地址协议不是`tcp + tls`，则忽略此选项。

`syslog-tls-cert`指定TLS证书文件的绝对路径。 如果地址协议不是`tcp + tls`，则忽略此选项。

`syslog-tls-key`指定TLS密钥文件的绝对路径。 如果地址协议不是`tcp + tls`，则忽略此选项。

`syslog-tls-skip-verify`配置TLS验证。 此验证是默认开启的，但可以通过将此选项设置为“true”来覆盖。
如果地址协议不是`tcp + tls`，则忽略此选项。

`tag`配置syslog日志中附加到APP-NAME后字符串。 默认情况下，Docker使用容器ID的前12个字符标记日志消息。 定制日志标记格式请参考[日志标签选项文档](log_tags.md) 。

`syslog-format`指定在记录时使用的syslog消息格式。 如果没有指定，默认为没有主机名的本地unix syslog格式规范。 指定rfc3164则以兼容RFC-3164的格式记录日志。 指定rfc5424以兼容RFC-5424的格式记录日志。 指定rfc5424micro以带微秒时间戳的兼容RFC-5424的格式记录日记。

`env`是逗号分隔的环境变量的键列表。 用于高级[日志标签选项](log_tags.md)。

`labels`是逗号分隔的标签键列表。 用于高级[日志标签选项](log_tags.md)

## journald选项

`journald`日志驱动程序将容器ID存储在日志的`CONTAINER_ID`字段中。 有关使用此日志驱动的详细信息，请参阅 [journald日志驱动](journald.md)参考文档。

## GELF选项

GELF日志驱动程序支持以下选项：

```no-highlight
--log-opt gelf-address=udp://host:port
--log-opt tag="database"
--log-opt labels=label1,label2
--log-opt env=env1,env2
--log-opt gelf-compression-type=gzip
--log-opt gelf-compression-level=1
```

`gelf-address`选项指定驱动程序连接的远程GELF服务器地址。 目前，只支持`udp`选项且
必须指定一个`port`值。 以下示例显示如何连接
`gelf`驱动程序到`192.168.0.42`上的端口为`12201`的GELF远程服务器

```bash
$ docker run -dit \
             --log-driver=gelf \
             --log-opt gelf-address=udp://192.168.0.42:12201 \
             alpine sh
```

默认情况下，Docker使用容器ID的前12个字符来标记日志。 自定义日志标记格式请参考[日志标签选项文档](log_tags.md) 。

`gelf`日志驱动程序支持`labels`和`env`选项。 它在`extra`字段上添加了额外键，前缀为
下划线（`_`）。
```json
// […]
"_foo": "bar",
"_fizz": "buzz",
// […]
```

`gelf-compression-type`选项可用于更改GELF驱动程序压缩日志消息的方式。 可接受的值是`gzip`，`zlib`和`none`。 默认情况下选择`gzip`。

当`gelf-compression-type`选择为`gzip`或`zlib`时，`gelf-compression-level`选项可以用来改变压缩级别。接受的值必须为-1到9（最高压缩）。 越高的值通常运行速度越慢，但压缩得更多。 默认值为1（最高速度）。

## Fluentd选项

您可以使用`--log-opt NAME=VALUE`选项来指定Fluentd日志驱动程序的其他选项。

- `fluentd-address`：指定`host：port`来连接[localhost：24224]
- `tag`：为`fluentd`消息指定标签
- `fluentd-buffer-limit`：指定fluentd日志缓冲区的最大大小[8MB]
- `fluentd-retry-wait`：连接重试之前的初始延迟（之后它按指数增加）[1000ms]
- `fluentd-max-retries`：docker突然失败之前连接重试的最大数量[1073741824]
- `fluentd-async-connect`：是否在初始连接时阻塞[false]

例如，要指定两个附加选项：

```bash
{% raw %}
$ docker run -dit \
             --log-driver=fluentd \
             --log-opt fluentd-address=localhost:24224 \
             --log-opt tag="docker.{{.Name}}" \
             alpine sh
{% endraw %}
```
如果容器无法连接到指定地址上的Fluentd守护程序且'fluentd-async-connect'没有启用，容器会立即停止。 

有关使用此日志记录驱动程序的详细信息，请参阅[fluentd日志驱动](fluentd.md)

## Amazon CloudWatch Logs选项

Amazon CloudWatch Logs日志记录驱动程序支持以下选项：

```none
{% raw %}
--log-opt awslogs-region=<aws_region>
--log-opt awslogs-group=<log_group_name>
--log-opt awslogs-stream=<log_stream_name>
--log-opt tag="docker.{{.Name}}" \
{% endraw %}
```

有关使用此日志记录驱动程序的详细信息，请参阅[awslogs日志驱动](awslogs.md)。

## Splunk选项

`splunk`日志记录驱动程序支持以下选项：

```none
--log-opt splunk-token=<splunk_http_event_collector_token>
--log-opt splunk-url=https://your_splunk_instance:8088
```

有关使用此日志记录驱动程序的详细信息，请参阅[splunk日志驱动](splunk.md)。

## ETW日志驱动选项

`etwlogs`日志驱动程序将每个日志消息作为ETW事件转发。 可以创建ETW侦听器以侦听这些事件。 这个驱动不接受任何选项。

ETW日志记录驱动程序仅在Windows上可用。 有关使用此日志记录驱动程序的详细信息，请参阅[etw日志驱动](etwlogs.md)。

## Google Cloud Logging选项

Google Cloud Logging驱动程式支持以下选项：

```none
--log-opt gcp-project=<gcp_projext>
--log-opt labels=<label1>,<label2>
--log-opt env=<envvar1>,<envvar2>
--log-opt log-cmd=true
```

有关使用Google Cloud Logging驱动程序的详细信息，请参阅[Google Cloud Logging驱动](gcplogs.md)。

## NATS日志选项

NATS日志驱动程式支持以下选项：

```none
--log-opt labels=<label1>,<label2>
--log-opt env=<envvar1>,<envvar2>
--log-opt tag=<tag>
--log-opt nats-servers="<comma separated list of nats servers uris>"
--log-opt nats-max-reconnect="<max attempts to connect to a server>"
--log-opt nats-subject="<subject where logs are sent>"
--log-opt nats-tls-ca-cert="<absolute path to cert>"
--log-opt nats-tls-cert="<absolute path to cert>"
--log-opt nats-tls-key="<absolute path to cert>"
--log-opt nats-tls-skip-verify="<value>"
```

详细信息，请参阅[NATS日志驱动](nats.md) 。

