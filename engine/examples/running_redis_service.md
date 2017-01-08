---
description: Installing and running a redis service
keywords: docker, example, package installation, networking,  redis
title: Dockerize a Redis service
---

Very simple, no frills, Redis service attached to a web application
using a link.

## 为 Redis 创建一个 Docker 容器

首先我们为新的 Redis 镜像创建一个 `Dockerfile` 。


```dockerfile
FROM        ubuntu:14.04
RUN         apt-get update && apt-get install -y redis-server
EXPOSE      6379
ENTRYPOINT  ["/usr/bin/redis-server"]
```

然后我们使用这个 `Dockerfile` 来构建一个镜像。
在下面的命令中使用您真实的用户名来替换 `<your username>` 。

```bash
$ docker build -t <your username>/redis .
```

## 启动 Redis 服务

使用我们刚才创建的镜像来启动容器，并把您的容器命名为 `redis`。

启动 Redis 服务，命令中使用 `-d` 参数，可以让容器在后台运行。

重要的是，我们没有把容器上的任何端口暴露给宿主机。取而代之的是采用容器连接的方式来对外提供 Redis 数据库的连接服务。


```bash
$ docker run --name redis -d <your username>/redis
```

## 创建您的网站应用容器

接下来我们可以给您的应用创建一个容器。我们会采用 `-link` 参数来创建一个容器连接，来对接刚才我们创建的别名是 `db` 的 `redis` 容器。采用这种方式会创建一个安全管道来连接 'redis' 容器，并且把该容器内运行的 Redis 实例只暴露给我们的应用容器。

```bash
$ docker run --link redis:db -i -t ubuntu:14.04 /bin/bash
```

在我们新创建的容器内部，我们需要安装 Redis 来获取 `redis-cli` 二进制命令行工具来测试 Redis 连接。

```bash
$ sudo apt-get update
$ sudo apt-get install redis-server
$ sudo service redis-server stop
```

由于我们使用了 `--link redis:db` 选项，所以 Docker 会在我们的网站应用容器内创建一些环境变量。

```bash
$ env | grep DB_

# 这里会返回一些类似的您的配置
DB_NAME=/violet_wolf/db
DB_PORT_6379_TCP_PORT=6379
DB_PORT=tcp://172.17.0.33:6379
DB_PORT_6379_TCP=tcp://172.17.0.33:6379
DB_PORT_6379_TCP_ADDR=172.17.0.33
DB_PORT_6379_TCP_PROTO=tcp
```

可以看到我们会获得一个小的、以 `DB` 为前缀的环境变量列表。`DB` 前缀来源于启动容器时所指定的容器连接别名。我们可以使用 `DB_PORT_6379_TCP_ADDR` 这个变量来连接 Redis 容器。
 
```bash
$ redis-cli -h $DB_PORT_6379_TCP_ADDR

$ redis 172.17.0.33:6379>
$ redis 172.17.0.33:6379> set docker awesome

OK

$ redis 172.17.0.33:6379> get docker

"awesome"

$ redis 172.17.0.33:6379> exit
```
我们可以使用网站应用容器内的这个环境变量或者其他的环境变量来便捷地创建出一个连接到 `redis` 容器的连接。
