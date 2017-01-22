---
description: CLI and log output formatting reference
keywords: format, formatting, output, templates, log
title: Format command and log output
---

容器使用 [Go 模板](https://golang.org/pkg/text/template/) 帮助用户定制某些命令和日志驱动的输出格式。每个命令都提供了一个详细
的在模板中支持的元素列表:

- [Docker Images formatting](../reference/commandline/images.md#formatting)
- [Docker Inspect formatting](../reference/commandline/inspect.md#examples)
- [Docker Log Tag formatting](logging/log_tags.md)
- [Docker Network Inspect formatting](../reference/commandline/network_inspect.md)
- [Docker PS formatting](../reference/commandline/ps.md#formatting)
- [Docker Stats formatting](../reference/commandline/stats.md#formatting)
- [Docker Volume Inspect formatting](../reference/commandline/volume_inspect.md)
- [Docker Version formatting](../reference/commandline/version.md#examples)

## 模板功能

容器提供一组基本方法操作模板元素。以下是一组可用方法的例子：


### `join`

`join'连接字符串列表成单个字符串。它在列表中的每个元素之间放置一个分隔符。

	{% raw %}
	$ docker ps --format '{{join .Names " or "}}'
	{% endraw %}

### `json`

`json` 讲一个元素转换成json格式.

	{% raw %}
	$ docker inspect --format '{{json .Mounts}}' container
	{% endraw %}

### `lower`

`lower` 将字符串转换成小写。

	{% raw %}
	$ docker inspect --format "{{lower .Name}}" container
	{% endraw %}

### `split`

`split` 将字符串切成由分隔符分隔的字符串列表。

	{% raw %}
	$ docker inspect --format '{{split (join .Names "/") "/"}}' container
  {% endraw %}

### `title`

`title` 将字符串首字母大写。

	{% raw %}
	$ docker inspect --format "{{title .Name}}" container
	{% endraw %}

### `upper`

`upper` 将字符串转成大写.

	{% raw %}
	$ docker inspect --format "{{upper .Name}}" container
	{% endraw %}
