---
description: How to set up and use certificates with a registry to verify access
keywords: Usage, registry, repository, client, root, certificate, docker, apache, ssl, tls, documentation, examples, articles, tutorials
redirect_from:
- /engine/articles/certificates/
title: 使用证书访问Docker Registry
---

在[使用HTTPS运行Docker](https.md)里，你已经了解到默认情况下Docker运行在Unix socket上，启用TLS可以让让Docker客户端和daemon通过HTTPS更安全的通信。TLS能保证registry认证和到registry之间的流量是加密的。

本文描述了如何保证Docker和registry之间传输的流量加密，并且配置基于证书的客户端服务器认证。

我们将会向你展示如何安装registry对应的证书认证机构(Certificate Authority, CA)证书，以及如何设置客户端TLS证书。

## 理解配置

要配置自定义证书，需要在`/etc/docker/certs.d`下创建一个和registry域名(比如`localhost`)相同的目录，目录下的所有`*.crt`文件都会被添加到CA根证书列表里。

> **注意:**
> 如果没有配置机构证书，Docker会使用系统默认根证书。

对Docker来说目录下成对的`<filename>.key/cert`意味着访问对应的repository时，用这两个文件作为客户端证书认证。

> **注意:**
> 如果目录里有多对证书，Docker会按照字母序逐个尝试每份证书。如果出现认证错误（例如: 403, 404, 5xx等），Docker会继续尝试下一个证书。

下面的是一个配置证书的例子：

```
    /etc/docker/certs.d/        <-- 证书目录
    └── localhost:5000          <-- Hostname:port（Hostname可以是域名）
       ├── client.cert          <-- 客户端证书
       ├── client.key           <-- 客户端证书的Key
       └── localhost.crt        <-- 颁发Registry证书的机构证书。

```

接下来的示例和操作系统相关，只是为了说明。你需要查询自己操作系统以创建对应操作系统的证书链。

## 创建客户端证书

首先你需要使用OpenSSL的`genrsa`和`req`命令生成一个RSA key，然后使用key创建证书。

    $ openssl genrsa -out client.key 4096
    $ openssl req -new -x509 -text -key client.key -out client.cert

> **注意:**
> 这些TLS命令只能在Linux上创建可用的证书。macOS上的OpenSSL版本和Docker需要的证书类型不兼容。

## 问题排查小贴士

Docker daemon把`.crt`文件当做CA证书，`.cert`文件当做客户端证书。如果CA证书不小心使用`.cert`作为扩展名，Docker daemon会输出下面的日志：

```
Missing key KEY_NAME for client certificate CERT_NAME. Note that CA certificates should use the extension .crt.
```

## 相关信息

* [使用可信镜像](index.md)
* [保护Docker daemon socket](https.md)
