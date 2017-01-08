---
description: Describes how to use the Splunk logging driver.
keywords: splunk, docker, logging, driver
redirect_from:
- /engine/reference/logging/splunk/
title: Splunk logging driver
---

`splunk`日志驱动程序将容器日志发送到Splunk Enterprise和Splunk Cloud的[HTTP事件收集器](http://dev.splunk.com/view/event-collector/SP-CAAAE6M)中。

## 用法

您可以通过设置`--log-driver`参数来给Docker守护进程配置默认日志驱动程序：

    dockerd --log-driver=splunk

您可以在`docker run`时使用`--log-driver`选项为特定容器设置日志驱动程序：

    docker run --log-driver=splunk ...
    
## Splunk选项

您可以使用`--log-opt NAME=VALUE`选项来指定Splunk日志驱动程序的其他选项：

|选项|是否必需|说明|
| ----------------------------- | ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `splunk-token` |必需| Splunk HTTP事件收集器标记。 |
| `splunk-url` |必需| Splunk Enterprise或Splunk Cloud实例的路径（包括HTTP事件收集器使用的端口和scheme）`https://your_splunk_instance8088`。 |
| `splunk-source` |可选|事件源。 |
| `splunk-sourcetype` |可选|事件源类型。 |
| `splunk-index` |可选|事件索引。 |
| `splunk-capath` |可选|根证书的路径。 |
| `splunk-caname` |可选|验证服务器证书所使用的名称;默认情况下将使用`splunk-url`的主机名。 |
| `splunk-insecureskipverify` |可选|忽略服务器证书验证。 |
| `splunk-format` |可选|消息格式。可以是`inline`，`json`或`raw`。默认为`inline`。 |
| `splunk-verify-connection` |可选|docker连接到Splunk服务器时验证。默认为true。 |
| `splunk-gzip` |可选|将事件发送到Splunk Enterprise或Splunk Cloud实例时启用/禁用gzip压缩以。默认为false。 |
| `splunk-gzip-level` |可选|设置gzip的压缩级别。有效值为-1（默认值），0（无压缩），1（最佳速度）... 9（最佳压缩）。默认为[DefaultCompression](https://golang.org/pkg/compress/gzip/#DefaultCompression). 。 |
| `tag` |可选|指定消息的标记，可以解释一些标签。默认值为 {% raw %}`{{.ID}}`{% endraw %}（容器ID的12个字符）。有关定制日志标记格式的信息，请参阅[日志标签选项文档](log_tags.md)。 |
| `labels` |可选|以逗号分隔的标签键列表，如果为容器指定这些标签，则应包含在消息中。 |
| `env` |可选|以逗号分隔的环境变量的键列表，如果为容器指定这些变量，则应包含在消息中。 |

如果`label`和`env`键之间有冲突，`env`的值优先。
两个选项都会向日志记录息的属性中添加其他字段。

以下是为Splunk Enterprise实例指定日志选项的示例。 实例安装Docker守护程序运行的机器上。 根证书路径和公用名使用HTTPS scheme指定。 `SplunkServerDefaultCert`是由Splunk证书自动生成。

    {% raw %}
    docker run --log-driver=splunk \
               --log-opt splunk-token=176FCEBF-4CF5-4EDF-91BC-703796522D20 \
               --log-opt splunk-url=https://splunkhost:8088 \
               --log-opt splunk-capath=/path/to/cert/cacert.pem \
               --log-opt splunk-caname=SplunkServerDefaultCert
               --log-opt tag="{{.Name}}/{{.FullID}}"
               --log-opt labels=location
               --log-opt env=TEST
               --env "TEST=false"
               --label location=west
           your/application
    {% endraw %}
    
### 消息格式

日志驱动程序默认以`inline`格式发送日志，每条日志都被内嵌为一个字符串，例如：

```
{
    "attrs": {
        "env1": "val1",
        "label1": "label1"
    },
    "tag": "MyImage/MyContainer",
    "source":  "stdout",
    "line": "my message"
}
{
    "attrs": {
        "env1": "val1",
        "label1": "label1"
    },
    "tag": "MyImage/MyContainer",
    "source":  "stdout",
    "line": "{\"foo\": \"bar\"}"
}
```

如果你的信息是JSON对象，你可能希望把它们内嵌到我们发送到Splunk的日志中。 指定`--log-opt splunk-format=json`则驱动程序会尽量以JSON对象的格式解析每行日志，并以内嵌对象发送。如果该行不能解析，则消息以`inline`格式发送。例如：

```
{
    "attrs": {
        "env1": "val1",
        "label1": "label1"
    },
    "tag": "MyImage/MyContainer",
    "source":  "stdout",
    "line": "my message"
}
{
    "attrs": {
        "env1": "val1",
        "label1": "label1"
    },
    "tag": "MyImage/MyContainer",
    "source":  "stdout",
    "line": {
        "foo": "bar"
    }
}
```

第三种是`raw`格式，通过`--log-opt splunk-format=raw`参数指定。属性（环境变量和标签）和标签会以前缀的形式加到消息前面。例如：

```
MyImage/MyContainer env1=val1 label1=label1 my message
MyImage/MyContainer env1=val1 label1=label1 {"foo": "bar"}
```

## 高级选项

Splunk日志驱动程序允许您通过为Docker守护程序指定下列环境变量来配置几个高级选项。

|环境变量名|默认值|说明|
| -------------------------------------------------- | --------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| `SPLUNK_LOGGING_DRIVER_POST_MESSAGES_FREQUENCY`| `5s` |如果没有批处理时，驱动程序发送消息的频率。您可以将此视为批量处理的最大等待时间。 |
| `SPLUNK_LOGGING_DRIVER_POST_MESSAGES_BATCH_SIZE`| `1000` |在批量发送消息之前，驱动程序最多等待多少条消息。 |
| `SPLUNK_LOGGING_DRIVER_BUFFER_MAX`| `10 * 1000` |如果驱动程序无法连接到远程服务器，它可以保存在缓冲区中的重试的最大消息量是。 |
| `SPLUNK_LOGGING_DRIVER_CHANNEL_SIZE`| `4 * 1000` |在送往批处理程序之前，在通道中可以缓存多少待处理消息。 |

