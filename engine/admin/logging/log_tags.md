---
description: Describes how to format tags for.
keywords: docker, logging, driver, syslog, Fluentd, gelf, journald
redirect_from:
- /engine/reference/logging/log_tags/
title: Log tags for logging driver
---

`tag`选项指定如何格式化容器的日志。缺省情况下，系统使用容器ID的前12个字符。要修改此行为，请指定`tag`选项：

```bash
$ docker run --log-driver=fluentd --log-opt fluentd-address=myhost.local:24224 --log-opt tag="mailer"
```

在指定某个标签的值时，Docker支持一些特殊的模板标记。

{% raw %}
| 标记 | 说明 |
| -------------------- | ------------------------------------------------------ |
| `{{.ID}}`|容器ID的前12个字符。 |
| `{{.FullID}}`|完整的容器ID。 |
| `{{.Name}}`|容器名称。 |
| `{{.ImageID}}`|容器的图片ID的前12个字符。 |
| `{{.ImageFullID}}`|容器镜像的完整标识。 |
| `{{.ImageName}}`|容器使用的镜像的名称。 |
| `{{.DaemonName}}`| docker程序的名称（`docker`）。 |
{% endraw %}

例如，指定{% raw %}`--log-opt tag="{{.ImageName}}/{{.Name}}/{{.ID}}"`{% endraw %} 选项会产生类似这样的`syslog `日志：

```none
Aug  7 18:33:19 HOSTNAME docker/hello-world/foobar/5790672ab6a0[9103]: Hello from Docker.
```

在启动时，系统会设置`container_name`字段并标签中设置{% raw %}`{{.Name}}`{% endraw %}。如果使用`docker rename`重命名容器，新名称不会反映在日志中。相反，这些日志会继续使用原始容器名称。

对于高级用法，生成的标记的使用[go模板](http://golang.org/pkg/text/template/)和容器的[日志上下文](https://github.com/docker/docker/blob/master/daemon/logger/context.go)。

作为使用syslog的一个例子，如果使用以下
命令，您将得到以下输出：

```bash
{% raw %}
$ docker run -it --rm \
    --log-driver syslog \
    --log-opt tag="{{ (.ExtraAttributes nil).SOME_ENV_VAR }}" \
    --log-opt env=SOME_ENV_VAR \
    -e SOME_ENV_VAR=logtester.1234 \
    flyinprogrammer/logtester
{% endraw %}
```

```none
Apr  1 15:22:17 ip-10-27-39-73 docker/logtester.1234[45499]: + exec app
Apr  1 15:22:17 ip-10-27-39-73 docker/logtester.1234[45499]: 2016-04-01 15:22:17.075416751 +0000 UTC stderr msg: 1
```

