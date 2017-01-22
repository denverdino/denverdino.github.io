---
description: Running and installing a PostgreSQL service
keywords: docker, example, package installation,  postgresql
title: Dockerize PostgreSQL
---

> **Note**:
> - **If you don't like sudo** then see [*Giving non-root
>   access*](../installation/binaries.md#giving-non-root-access)

## 在 Docker 上安装 PostgreSQL

如果在 [Docker Hub](http://hub.docker.com) 上面没有镜像能够满足您需求，您可以自己制作一个镜像。

第一步，创建一个新的 `Dockerfile` :

> **Note**:
本篇  PostgreSQL 引导文档只适用于开发环境。请参考 PostgreSQL 的文档来调整您的配置文件，以确保 PostgreSQL 在线上是有安全保障的。

```dockerfile
#
# Dockerfile 示例文档地址 https://docs.docker.com/examples/postgresql_service/
#

FROM ubuntu
MAINTAINER SvenDowideit@docker.com

# 添加 PostgreSQL 的PGP键以验证其 Debian 安装包, 
# 其值应该和 https://www.postgresql.org/media/keys/ACCC4CF8.asc 一样
RUN apt-key adv --keyserver hkp://p80.pool.sks-keyservers.net:80 --recv-keys B97B0AFCAA1A47F044F244A07FCC7D46ACCC4CF8

# 添加 PostgreSQL's 仓库. 该仓库包涵了PostgreSQL 最新的稳定版本``9.3``.
RUN echo "deb http://apt.postgresql.org/pub/repos/apt/ precise-pgdg main" > /etc/apt/sources.list.d/pgdg.list

# 安装 ``python-software-properties``, ``software-properties-common`` 和 PostgreSQL 9.3
# 在打包镜像期间会有一些告警信息（红色字体）显示出来。
# 您可以通过在每个apt-get 命令之前添加前缀 DEBIAN_FRONTEND=noninteractive 来隐藏告警信息
RUN apt-get update && apt-get install -y python-software-properties software-properties-common postgresql-9.3 postgresql-client-9.3 postgresql-contrib-9.3

# 提示: 官方的 Debian and Ubuntu 镜像会在每个 ``apt-get``命令之后自动执行 ``apt-get clean`` 命令 

# 其他的命令以用户 ``postgres`` 的身份来运行。
# 该用户是在使用命令 ``apt-get installed`` 安装 ``postgres-9.3`` 安装包时创建的
USER postgres

# 创建一个 用户名、密码都是 ``docker`` 的 PostgreSQL 角色，然后再创建一个由该角色拥有的数据库 `docker`。
# 注意: 这里我们使用符号 ``&&\`` 来链接每一个命令，因为字符 ``\``允许将一条 RUN 命令拆分为多行显示
RUN    /etc/init.d/postgresql start &&\
    psql --command "CREATE USER docker WITH SUPERUSER PASSWORD 'docker';" &&\
    createdb -O docker docker

# 请调整 PostgreSQL 的配置，以便能够远程连接数据库。
RUN echo "host all  all    0.0.0.0/0  md5" >> /etc/postgresql/9.3/main/pg_hba.conf

# 给 ``/etc/postgresql/9.3/main/postgresql.conf`` 配置文件添加 ``listen_addresses`` 地址监听配置项
RUN echo "listen_addresses='*'" >> /etc/postgresql/9.3/main/postgresql.conf

# 对外暴露 PostgreSQL 端口
EXPOSE 5432

# 添加数据卷来备份配置文件、日志文件和数据库实例
VOLUME  ["/etc/postgresql", "/var/log/postgresql", "/var/lib/postgresql"]

# 配置容器运行时需要默认执行的命令、脚本
CMD ["/usr/lib/postgresql/9.3/bin/postgres", "-D", "/var/lib/postgresql/9.3/main", "-c", "config_file=/etc/postgresql/9.3/main/postgresql.conf"]

```

使用刚才的 Dockerfile 来构建镜像，并且给它指定一个名字。

```bash
$ docker build -t eg_postgresql .
```

启动 PostgreSQL 服务容器 (前台运行):

```bash
$ docker run --rm -P --name pg_test eg_postgresql
```

有两种方式可以连接到 PostgreSQL 服务器. 我们可以使用 [*Link
Containers*](../userguide/networking/default_network/dockerlinks.md), 或者我们可以通过自己的主机连接（或者通过网络）。


> **注意**: 参数 `--rm` 会在容器成功退出后删除容器和它对应的镜像。

### 使用 container link

在客户端的 `docker run` 命令中使用参数 `-link remote_name:local_alias` 可以让一个容器直接连接到另外一个容器的端口上。该参数会设置两个容器连接时需要使用的一系列的环境变量：

```bash
$ docker run --rm -t -i --link pg_test:pg eg_postgresql bash

postgres@7ef98b1b7243:/$ psql -h $PG_PORT_5432_TCP_ADDR -p $PG_PORT_5432_TCP_PORT -d docker -U docker --password
```

### 从您的主机操作系统上连接

如果您已经安装了postgresql的客户端, 您也可以使用主机映射的端口来做连接测试。首先，您需要使用命令`docker ps` 来查看容器映射到宿主机的具体的端口号，然后再使用客户端命令连接：

```bash
$ docker ps

CONTAINER ID        IMAGE                  COMMAND                CREATED             STATUS              PORTS                                      NAMES
5e24362f27f6        eg_postgresql:latest   /usr/lib/postgresql/   About an hour ago   Up About an hour    0.0.0.0:49153->5432/tcp                    pg_test

$ psql -h localhost -p 49153 -d docker -U docker --password
```

### 测试数据库

当您通过了权限验证并且界面出现了 `docker =#` 输入提示, 您就可以创建一个数据表并且存数据了。

```sql
psql (9.3.1)
Type "help" for help.

$ docker=# CREATE TABLE cities (
docker(#     name            varchar(80),
docker(#     location        point
docker(# );
CREATE TABLE
$ docker=# INSERT INTO cities VALUES ('San Francisco', '(-194.0, 53.0)');
INSERT 0 1
$ docker=# select * from cities;
     name      | location
---------------+-----------
 San Francisco | (-194,53)
(1 row)
```

### 使用容器数据卷

您可以使用具名数据卷来查看 PostgreSQL的日志文件，也可以备份您的配置文件和数据：

```bash
$ docker run --rm --volumes-from pg_test -t -i busybox sh

/ # ls
bin      etc      lib      linuxrc  mnt      proc     run      sys      usr
dev      home     lib64    media    opt      root     sbin     tmp      var
/ # ls /etc/postgresql/9.3/main/
environment      pg_hba.conf      postgresql.conf
pg_ctl.conf      pg_ident.conf    start.conf
/tmp # ls /var/log
ldconfig    postgresql
```
