---
description: Using the Ambassador pattern to abstract (network) services
keywords: Examples, Usage, links, docker, documentation, examples, names, name,  container naming
redirect_from:
- /engine/articles/ambassador_pattern_linking/
title: Link via an ambassador container
---

Docker鼓励支持服务的可移植性，而非以硬编码的方式在服务提供者和消费者之间定义网络连接。举例来说，按照下面的模式:

    (consumer) --> (redis)

如果要连接不同的`redis` 服务，则需要重启`consumer`。要替代这种模式，可以添加ambassador对象。如下所示:

    (consumer) --> (redis-ambassador) --> (redis)

或者

    (consumer) --> (redis-ambassador) ---network---> (redis-ambassador) --> (redis)

当要求consumer重新连接一个不同的Redis服务器时，只需重启与consumer相连的 `redis-ambassador` 容器即可。

这种模式同样允许透明地将Redis服务器从consumer所在主机迁移到其他主机。

使用 `svendowideit/ambassador` 容器，连接的定义完全由 `docker run` 命令的参数来控制。


## 使用两台主机的示例

在一台Docker主机上启动Redis服务器。

    big-server $ docker run -d --name redis crosbymichael/redis

添加一个ambassador与Redis服务器连接，并定义对外的端口映射。

    big-server $ docker run -d --link redis:redis --name redis_ambassador -p 6379:6379 svendowideit/ambassador

在第二台主机上，创建另一个ambassador。并为每个需要被访问的 `big-server` 上的远程端口设置环境变量。

    client-server $ docker run -d --name redis_ambassador --expose 6379 -e REDIS_PORT_6379_TCP=tcp://192.168.1.52:6379 svendowideit/ambassador

在 `client-server` 这台主机上，只需定义Redis客户端容器与本机Redis ambassador间的连接，就可以访问远程的Redis服务了。

    client-server $ docker run -i -t --rm --link redis_ambassador:redis relateiq/redis-cli
    redis 172.17.0.160:6379> ping
    PONG

## 如何工作

下面的例子展示了 `svendowideit/ambassador` 容器自动完成的工作（包括了少量的 `sed` 操作）。

在Redis服务运行的Docker主机（192.168.1.52）上：

    # 启动redis服务
    $ docker run -d --name redis crosbymichael/redis

    # 拉取redis-cli镜像以进行连接测试
    $ docker pull relateiq/redis-cli

    # 通过直接访问测试redis服务
    $ docker run -t -i --rm --link redis:redis relateiq/redis-cli
    redis 172.17.0.136:6379> ping
    PONG
    ^D

    # 添加redis ambassador
    $ docker run -t -i --link redis:redis --name redis_ambassador -p 6379:6379 alpine:3.2 sh

在 `redis_ambassador` 容器中，可以查看到与Redis容器连接相关的环境变量 `env`：

    / # env
    REDIS_PORT=tcp://172.17.0.136:6379
    REDIS_PORT_6379_TCP_ADDR=172.17.0.136
    REDIS_NAME=/redis_ambassador/redis
    HOSTNAME=19d7adf4705e
    SHLVL=1
    HOME=/root
    REDIS_PORT_6379_TCP_PORT=6379
    REDIS_PORT_6379_TCP_PROTO=tcp
    REDIS_PORT_6379_TCP=tcp://172.17.0.136:6379
    TERM=xterm
    PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
    PWD=/
    / # exit

ambassador容器中的 `socat` 脚本利用这些环境变量将Redis服务暴露出来（通过端口映射 `-p 6379:6379` ）：

    $ docker rm redis_ambassador
    $ CMD="apk update && apk add socat && sh"
    $ docker run -t -i --link redis:redis --name redis_ambassador -p 6379:6379 alpine:3.2 sh -c "$CMD"
    [...]
    / # socat -t 100000000 TCP4-LISTEN:6379,fork,reuseaddr TCP4:172.17.0.136:6379

现在可以通过ambassador来连接Redis服务了：

在另一台主机上运行：

    $ CMD="apk update && apk add socat && sh"
    $ docker run -t -i --expose 6379 --name redis_ambassador alpine:3.2 sh -c "$CMD"
    [...]
    / # socat -t 100000000 TCP4-LISTEN:6379,fork,reuseaddr TCP4:192.168.1.52:6379

然后拉取 `redis-cli` 镜像，并借助ambassador桥接访问redis服务。

    $ docker pull relateiq/redis-cli
    $ docker run -i -t --rm --link redis_ambassador:redis relateiq/redis-cli
    redis 172.17.0.160:6379> ping
    PONG

## svendowideit/ambassador镜像的 Dockerfile

 `svendowideit/ambassador`基于 `alpine:3.2` 镜像，并安装了 `socat` 。启动容器时，会运行一小段 `sed` 脚本来解析与连接相关的环境变量（可能有多个），并以此建立端口转发。在远程主机上，需要通过 `-e` 命令行选项来设置这些环境变量。

    --expose 1234 -e REDIS_PORT_1234_TCP=tcp://192.168.1.52:6379

这将会把对本地端口 `1234` 的访问转发到远程的IP地址和端口，比如这里的 `192.168.1.52:6379`。

    #
    # 构建镜像
    #   docker build -t svendowideit/ambassador .
    # 运行该容器（在服务后端运行的主机上）
    #   docker run -t -i -link redis:redis -name redis_ambassador -p 6379:6379 svendowideit/ambassador
    # 在远程主机上，建立另一个ambassador
    #    docker run -t -i -name redis_ambassador -expose 6379 -e REDIS_PORT_6379_TCP=tcp://192.168.1.52:6379 svendowideit/ambassador sh
    # 可以在后面的链接中获取更多关于此过程的信息 https://docs.docker.com/articles/ambassador_pattern_linking/

    # 使用alpine镜像是因为其体积最小，而且自带包管理器 
    # 这里仅需一个能够运行 env 和 socat（或者其他具有同样能力的命令）的容器就足够了
    FROM	alpine:3.2
    MAINTAINER	SvenDowideit@home.org.au

    RUN apk update && \
    	apk add socat && \
    	rm -r /var/cache/

    CMD	env | grep _TCP= | (sed 's/.*_PORT_\([0-9]*\)_TCP=tcp:\/\/\(.*\):\(.*\)/socat -t 100000000 TCP4-LISTEN:\1,fork,reuseaddr TCP4:\2:\3 \&/' && echo wait) | sh
