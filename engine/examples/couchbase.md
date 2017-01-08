---
description: Dockerizing a Couchbase service
keywords: docker, example, package installation, networking, couchbase
title: 容器化一个 Couchbase 服务
---

本章将展现如何用Docker Compose启动一个[Couchbase](http://couchbase.com) 服务， 通过它的[REST
API](http://developer.couchbase.com/documentation/server/4.0/rest-api/rest-endpoints-all.html)配置它,并且查询。

Couchbase 是一个开源的, 面向文档型NoSQL数据库，用于现代web，手机和物联网应用。它设计致力于易部署，大规模服务性能。

## 启动 Couchbase 服务

Couchbase 的Docker 镜像已在 [Docker
Hub](https://hub.docker.com/_/couchbase/)发布.

启动 Couchbase 服务:

```bash
$ docker run -d --name db -p 8091-8093:8091-8093 -p 11210:11210 couchbase
```

每个暴露端口的设计意义 [Couchbase 开发者入口 -
网络配置](http://developer.couchbase.com/documentation/server/4.1/install/install-ports.html).

日志可以用 `docker logs` 命令看到:

```bash
$ docker logs db

启动 Couchbase 服务 -- 想要用它的Web的UI界面可以访问 http://<ip>:8091
```

> **Note**: 本文的例子假设Docker Host是`192.168.99.100`，您可以用您Docker Host的真实IP地址替换`192.168.99.100`。如果您在Docker Machine上运行，您可以运行`docker-machine ip <MACHINE-NAME>`获得到您Docker Host的IP地址。

日志显示， Couchbase 控制台可以通过`http://192.168.99.100:8091` 访问，默认的用户名是 `Administrator` ，密码是`password`。

## 配置 Couchbase Docker 容器

默认情况下，Couchbase 服务在使用前需要通过控制台配置。 配置过程可以通过REST API简化

### 配置存数据的内存以及索引服务

Couchbase可以配置Data，查询，和索引三个不同的服务，每个服务有不同的操作需求，例如查询是一个CPU强需求操作，所以需要一个更快的处理器，索引主要在磁盘进行操作，所以需要一个更快的固态硬盘。Data需要快速的读/写，所以需要更多内存。

内存只有在Data和Query服务中需要被配置。

```bash
$ curl -v -X POST http://192.168.99.100:8091/pools/default -d memoryQuota=300 -d indexMemoryQuota=300

* Hostname was NOT found in DNS cache
*   Trying 192.168.99.100...
* Connected to 192.168.99.100 (192.168.99.100) port 8091 (#0)

> POST /pools/default HTTP/1.1
> User-Agent: curl/7.37.1
> Host: 192.168.99.100:8091
> Accept: */*
> Content-Length: 36
> Content-Type: application/x-www-form-urlencoded
>
* upload completely sent off: 36 out of 36 bytes
< HTTP/1.1 401 Unauthorized
< WWW-Authenticate: Basic realm="Couchbase Server Admin / REST"
* Server Couchbase Server is not blacklisted
< Server: Couchbase Server
< Pragma: no-cache
< Date: Wed, 25 Nov 2015 22:48:16 GMT
< Content-Length: 0
< Cache-Control: no-cache
<
* Connection #0 to host 192.168.99.100 left intact
```

这个命令显示了一个向`/pools/default`发送的HTTP
 POST请求。
host是Docker Machine的IP地址，port是Couchbase服务暴露的端口。请求中也传入了服务所需的内存和索引配额。

### 配置Data, Query, and Index 服务

每个实例上可以配置任意服务，无论是三个，还是只有一个。 
这使得每个服务可以定制使用和启动。比如，如果一个Docker宿主机运行在一个有固态硬盘的机器上，Data服务将被启动。

```bash
$ curl -v http://192.168.99.100:8091/node/controller/setupServices -d 'services=kv%2Cn1ql%2Cindex'

* Hostname was NOT found in DNS cache
*   Trying 192.168.99.100...
* Connected to 192.168.99.100 (192.168.99.100) port 8091 (#0)

> POST /node/controller/setupServices HTTP/1.1
> User-Agent: curl/7.37.1
> Host: 192.168.99.100:8091
> Accept: */*
> Content-Length: 26
> Content-Type: application/x-www-form-urlencoded
>
* upload completely sent off: 26 out of 26 bytes
< HTTP/1.1 200 OK
* Server Couchbase Server is not blacklisted
< Server: Couchbase Server
< Pragma: no-cache
< Date: Wed, 25 Nov 2015 22:49:51 GMT
< Content-Length: 0
< Cache-Control: no-cache
<
* Connection #0 to host 192.168.99.100 left intact
```

这个命令是一个发送给`/node/controller/setupServices`的HTTP POST请求。 这个命令中，所有三个服务都被配置到Couchbase服务中，Data服务标识为`kv`，查询服务标识为`n1ql`，索引服务标识为`index`。

### 启动Couchbase服务的证书

设置用户名和密码验证，帐号密码立即生效，用于管理Couchbase服务。 

```
curl -v -X POST http://192.168.99.100:8091/settings/web -d port=8091 -d username=Administrator -d password=password
* Hostname was NOT found in DNS cache
*   Trying 192.168.99.100...
* Connected to 192.168.99.100 (192.168.99.100) port 8091 (#0)
> POST /settings/web HTTP/1.1
> User-Agent: curl/7.37.1
> Host: 192.168.99.100:8091
> Accept: */*
> Content-Length: 50
> Content-Type: application/x-www-form-urlencoded
>
* upload completely sent off: 50 out of 50 bytes
< HTTP/1.1 200 OK
* Server Couchbase Server is not blacklisted
< Server: Couchbase Server
< Pragma: no-cache
< Date: Wed, 25 Nov 2015 22:50:43 GMT
< Content-Type: application/json
< Content-Length: 44
< Cache-Control: no-cache
<
* Connection #0 to host 192.168.99.100 left intact
{"newBaseUri":"http://192.168.99.100:8091/"}
```

这个命令是一个发送给 `/settings/web` 的HTTP POST请求，请求中包含用户名和密码。

### 加入样本数据

Couchbase服务可以被简单地在Couchbase实例中加入一些样本数据。

```
curl -v -u Administrator:password -X POST http://192.168.99.100:8091/sampleBuckets/install -d '["travel-sample"]'
* Hostname was NOT found in DNS cache
*   Trying 192.168.99.100...
* Connected to 192.168.99.100 (192.168.99.100) port 8091 (#0)
* Server auth using Basic with user 'Administrator'
> POST /sampleBuckets/install HTTP/1.1
> Authorization: Basic QWRtaW5pc3RyYXRvcjpwYXNzd29yZA==
> User-Agent: curl/7.37.1
> Host: 192.168.99.100:8091
> Accept: */*
> Content-Length: 17
> Content-Type: application/x-www-form-urlencoded
>
* upload completely sent off: 17 out of 17 bytes
< HTTP/1.1 202 Accepted
* Server Couchbase Server is not blacklisted
< Server: Couchbase Server
< Pragma: no-cache
< Date: Wed, 25 Nov 2015 22:51:51 GMT
< Content-Type: application/json
< Content-Length: 2
< Cache-Control: no-cache
<
* Connection #0 to host 192.168.99.100 left intact
[]
```

这个命令是一个发送给 `/sampleBuckets/install` 的HTTP POST请求，请求中包含样本bucket的名字。

恭喜你，你现在已经运行了一个Couchbase容器，它完全用REST API完成配置。

## Query Couchbase using CBQ

[CBQ](http://developer.couchbase.com/documentation/server/4.1/cli/cbq-tool.html),
是Couchbase Query的缩写, 是一个命令行工具，支持用JSON格式进行对Couchbase服务的create，read，update和delete操作。 这个工具在Couchbase的Docker镜像中已被默认安装。

运行 CBQ 工具:

```
docker run -it --link db:db couchbase cbq --engine http://db:8093
Couchbase query shell connected to http://db:8093/ . Type Ctrl-D to exit.
cbq>
```

CBQ的 `--engine` 参数允许指定运行在Docker宿主机上的Couchbase服务的host和port。 对host参数，通常是Couchbase服务所运行的主机的主机名，或者IP地址。 在本例中，启动服务时的容器名，`db`，可被用于host。 `8093`端口监听所有进入的查询。

Couchbase支持用[N1QL](http://developer.couchbase.com/documentation/server/4.1/n1ql/n1ql-language-reference/index.html)查询JSON文档。
N1QL 是一个综合性，声明式的查询语言，给JSON文档，提供了类SQL的查询能力。

通过执行一个N1QL查询数据库：

```
cbq> select * from `travel-sample` limit 1;
{
    "requestID": "97816771-3c25-4a1d-9ea8-eb6ad8a51919",
    "signature": {
        "*": "*"
    },
    "results": [
        {
            "travel-sample": {
                "callsign": "MILE-AIR",
                "country": "United States",
                "iata": "Q5",
                "icao": "MLA",
                "id": 10,
                "name": "40-Mile Air",
                "type": "airline"
            }
        }
    ],
    "status": "success",
    "metrics": {
        "elapsedTime": "60.872423ms",
        "executionTime": "60.792258ms",
        "resultCount": 1,
        "resultSize": 300
    }
}
```

## Couchbase Web Console

[Couchbase Web
控制台](http://developer.couchbase.com/documentation/server/4.1/admin/ui-intro.html)
是一个用于管理Couchbase实例的Web控制台。它可以用这个地址访问：

`http://192.168.99.100:8091/`

将 IP 换为你的Docker Machine的IP地址，假如你是在本地运行Docker，则替换为 `localhost`。

![Couchbase Web 控制台](couchbase/web-console.png)
