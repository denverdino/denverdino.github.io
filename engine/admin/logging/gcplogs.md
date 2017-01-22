---
description: Describes how to use the Google Cloud Logging driver.
keywords: gcplogs, google, docker, logging, driver
title: Google Cloud Logging driver
---

Google Cloud Logging驱动程序会将容器日志发送到
<a href="https://cloud.google.com/logging/docs/" target="_blank">Google Cloud Logging</a>。

## 用法

您可以通过设置`--log-driver'参数来给Docker守护进程配置默认日志驱动程序：

    dockerd --log-driver=gcplogs

您可以在`docker run`时使用`--log-driver`选项为特定容器设置日志驱动程序：

    docker run --log-driver=gcplogs ...

这个日志驱动程序没有实现日志阅读器，所以它不兼容`docker logs`。

如果Docker检测到它正在Google Cloud Project中运行，它会从
<a href="https://cloud.google.com/compute/docs/metadata" target="_blank">实例元数据服务</a>
中获取配置。

否则，用户必须用`--gcp-project`选项来标明把日志存储到哪个项目中，而且Docker会尝试从
<a href="https://developers.google.com/identity/protocols/application-default-credentials" target="_blank">Google应用程式预设凭证</a>
中获取凭证信息。 

`--gcp-project`参数优先于从元数据服务器发现的信息，因此在Goo:le Cloud Project中运行的Docker守护程序可以用该参数将日志记录到另外的项目中。

Docker从Google云元数据服务器中获取区域，实例名称和实例ID的值。
如果元数据服务器不可用，这些值可以通过参数提供。参数不会覆盖从元数据服务器中获取的值。

## gcplogs选项

您可以使用`--log-opt NAME=VALUE`参数指定Google Logging驱动程序的其他选项：

| 选项 | 是否必需 | 说明 |
| ----------------------------- | ---------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `gcp-project` |可选|日志记录到哪个GCP项目中。默认从GCE元数据服务发现此值。 |
| `gcp-log-cmd` |可选|是否记录容器的启动命令。默认为false。|
| `labels` |可选|以逗号分隔的标签名称列表，如果为容器指定这些标签，则应包含在消息中。 |
| `env` |可选|以逗号分隔的环境变量的名称列表，如果为容器指定这些变量，则应包含在消息中。 |
| `gcp-meta-zone` |可选|实例所在区域的名称。 |
| `gcp-meta-name` |可选|实例名称。 |
| `gcp-meta-id`|可选|实例ID。 |

如果`label`和`env`名称之间有冲突，`env`的值优先。两个选项都向日志消息中添加附加字段。

以下是日志选项的一个示例，该示例发送日志到通过GCE元数据服务器获得的默认日志记录地。

    docker run --log-driver=gcplogs \
        --log-opt labels=location \
        --log-opt env=TEST \
        --log-opt gcp-log-cmd=true \
        --env "TEST=false" \
        --label location=west \
        your/application

该配置还指示驱动程序在日志中包含标签`location`，环境变量`ENV`，以及容器的启动命令。

下面是使用外部GCE的参数示例（守护程序必须已配置了GOOGLE_APPLICATION_CREDENTIALS选项）：

    docker run --log-driver=gcplogs \
        --log-opt gcp-project=test-project
        --log-opt gcp-meta-zone=west1 \
        --log-opt gcp-meta-name=`hostname` \
        your/application

