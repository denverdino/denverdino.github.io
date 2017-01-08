---
description: Sharing data between 2 couchdb databases
keywords: docker, example, package installation, networking, couchdb,  data volumes
title: 容器内部署一个 CouchDB 服务
---

> **Note**:
> - **如果你不喜欢sudo** 请看 [*使用非root
>   权限*](../installation/binaries.md#giving-non-root-access)

我们会看到在两个CouchDB容器中使用数据卷同一份数据的例子，可以用来测试相同数据不同版本CouchDB的热升级。

## 创建第一个数据库

注意，我们创建`/var/lib/couchdb`目录作为数据据。

    $ COUCH1=$(docker run -d -p 5984 -v /var/lib/couchdb shykes/couchdb:2013-05-03)

## 在第一个数据库中加入数据

我们假设你的docker主机可以通过`localhost`访问，如果不行，将`localhost`替换为你的docker主机的公网IP。

    $ HOST=localhost
    $ URL="http://$HOST:$(docker port $COUCH1 5984 | grep -o '[1-9][0-9]*$')/_utils/"
    $ echo "Navigate to $URL in your browser, and use the couch interface to add data"

## 创建第二个数据库

这次，我们会请求`$COUCH1`数据卷的共享权限

    $ COUCH2=$(docker run -d -p 5984 --volumes-from $COUCH1 shykes/couchdb:2013-05-03)

## 在第二个数据库中浏览数据

    $ HOST=localhost
    $ URL="http://$HOST:$(docker port $COUCH2 5984 | grep -o '[1-9][0-9]*$')/_utils/"
    $ echo "Navigate to $URL in your browser. You should see the same data as in the first database"'!'

恭喜，现在你已经启动了两个Couchdb容器，除了数据外，其他部分完全相互独立。
