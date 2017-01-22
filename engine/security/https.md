---
description: How to setup and run Docker with HTTPS
keywords: docker, docs, article, example, https, daemon, tls, ca,  certificate
redirect_from:
- /engine/articles/https/
- /articles/https/
title: 保护Docker daemon
---

默认情况下，Docker通过Unix socket提供服务，也可以选择使用HTTP提供服务。

如果你需要安全通过网络访问Docker，可以用`tlsverify`参数启用TLS，`tlscacert`指向受信任的CA证书。

对于daemon，只允许通过使用CA签发的证书的客户端访问。在客户端，只能连接配置了CA签发的证书的服务器。

> **警告**:
> 使用TLS和管理CA属于高级主题。在生产环境使用之前请请先熟悉OpenSSL，x509和TLS

> **警告**:
> 这些TLS命令只在Linux上才能创建证书，macOS上的OpenSSL版本和Docker需要的证书不兼容。

## 使用OpenSSL创建CA、服务器和客户端密钥

> **注意**: 用你自己Docker daemon主机的DNS名替换下面例子中的`$HOST`

首先创建CA公钥和私钥

    $ openssl genrsa -aes256 -out ca-key.pem 4096
    Generating RSA private key, 4096 bit long modulus
    ............................................................................................................................................................................................++
    ........++
    e is 65537 (0x10001)
    Enter pass phrase for ca-key.pem:
    Verifying - Enter pass phrase for ca-key.pem:
    $ openssl req -new -x509 -days 365 -key ca-key.pem -sha256 -out ca.pem
    Enter pass phrase for ca-key.pem:
    You are about to be asked to enter information that will be incorporated
    into your certificate request.
    What you are about to enter is what is called a Distinguished Name or a DN.
    There are quite a few fields but you can leave some blank
    For some fields there will be a default value,
    If you enter '.', the field will be left blank.
    -----
    Country Name (2 letter code) [AU]:
    State or Province Name (full name) [Some-State]:Queensland
    Locality Name (eg, city) []:Brisbane
    Organization Name (eg, company) [Internet Widgits Pty Ltd]:Docker Inc
    Organizational Unit Name (eg, section) []:Sales
    Common Name (e.g. server FQDN or YOUR name) []:$HOST
    Email Address []:Sven@home.org.au

现在，我们有了CA，你可以创建服务器密钥和证书CSR，要确保"Common Name" (例子中的服务器FQDN或者你的名字)匹配要访问Docker所用的hostname：

> **注意**: 用你自己Docker daemon主机的DNS名替换下面例子中的`$HOST`

    $ openssl genrsa -out server-key.pem 4096
    Generating RSA private key, 4096 bit long modulus
    .....................................................................++
    .................................................................................................++
    e is 65537 (0x10001)
    $ openssl req -subj "/CN=$HOST" -sha256 -new -key server-key.pem -out server.csr

接下来，我们要用CA给公钥签名：

因为TLS连接既可以通过DNS名也可以通过IP地址，创建证书的时候要指定。例如下面的连接使用`10.10.10.20`和`127.0.0.1`：

    $ echo subjectAltName = DNS:$HOST,IP:10.10.10.20,IP:127.0.0.1 > extfile.cnf

    $ openssl x509 -req -days 365 -sha256 -in server.csr -CA ca.pem -CAkey ca-key.pem \
      -CAcreateserial -out server-cert.pem -extfile extfile.cnf
    Signature ok
    subject=/CN=your.host.com
    Getting CA Private Key
    Enter pass phrase for ca-key.pem:

为了实现客户端认证，创建一个客户端密钥和CSR：

    $ openssl genrsa -out key.pem 4096
    Generating RSA private key, 4096 bit long modulus
    .........................................................++
    ................++
    e is 65537 (0x10001)
    $ openssl req -subj '/CN=client' -new -key key.pem -out client.csr

为了让证书适用于客户端认证，创建一个扩展配置文件：

    $ echo extendedKeyUsage = clientAuth > extfile.cnf

签名公钥：

    $ openssl x509 -req -days 365 -sha256 -in client.csr -CA ca.pem -CAkey ca-key.pem \
      -CAcreateserial -out cert.pem -extfile extfile.cnf
    Signature ok
    subject=/CN=client
    Getting CA Private Key
    Enter pass phrase for ca-key.pem:

生成`cert.pem`和`server-cert.pem`之后就可以安全的删除两个证书csr文件了：

    $ rm -v client.csr server.csr

默认的`umask`是022，所有人都可以读取，你和同一个组的用户可以写。

为了保护密钥不受破坏，取消密钥的写权限，只有自己可读：

    $ chmod -v 0400 ca-key.pem key.pem server-key.pem

证书所有人可读，但是你要去掉写权限，以免证书被破坏：

    $ chmod -v 0444 ca.pem server-cert.pem cert.pem

现在，你可以让Docker daemon只接受具有受信CA颁发的证书的客户端：

    $ dockerd --tlsverify --tlscacert=ca.pem --tlscert=server-cert.pem --tlskey=server-key.pem \
      -H=0.0.0.0:2376

为了连接Docker并验证证书，你需要提供你的客户端密钥、证书和受信CA：

> **注意**: 用你自己Docker daemon主机的DNS名替换下面例子中的`$HOST`

    $ docker --tlsverify --tlscacert=ca.pem --tlscert=cert.pem --tlskey=key.pem \
      -H=$HOST:2376 version

> **注意**: Docker+TLS要运行在TCP端口2376说

> **警告**:
> 在上面的例子中，使用证书的情况下，你不需要用sudo运行docker命令，也不需要把用户加到`docker`组里。也就是说每个拥有key的人都可以给Docker daemon发送指令，具有访问宿主机的root权限。要用你的root密码保护这些key

## 默认的安全行为

如果你在默认情况下保证Docker客户端连接的安全性，可以把证书文件放到`~/.docker`目录下，设置`DOCKER_HOST` 和 `DOCKER_TLS_VERIFY` 变量（而不是每次调用传参数 `-H=tcp://$HOST:2376` 和 `--tlsverify` ）

    $ mkdir -pv ~/.docker
    $ cp -v {ca,cert,key}.pem ~/.docker
    $ export DOCKER_HOST=tcp://$HOST:2376 DOCKER_TLS_VERIFY=1

Docker现在默认使用安全连接：

    $ docker ps

## 其他模式

如果你不想用完整的双路认证，也可以通过参数组合让Docker运行在其他模式下：

### Daemon模式

 - `tlsverify`, `tlscacert`, `tlscert`, `tlskey` set: 认证客户端
 - `tls`, `tlscert`, `tlskey`: 不认证客户端

### Client模式

 - `tls`: 基于公共/默认CA池认证服务器
 - `tlsverify`, `tlscacert`: 使用指定的CA验证服务器
 - `tls`, `tlscert`, `tlskey`: 使用客户端证书，不验证服务器
 - `tlsverify`, `tlscacert`, `tlscert`, `tlskey`: 使用客户端证书同时使用给定的CA验证服务器。

只要存在客户端就会发送它的证书，所以你只需要把key放到`~/.docker/{ca,cert,key}.pem`，或者，如果你想在其他地方存放key，你可以用环境变量`DOCKER_CERT_PATH`指定路径。

    $ export DOCKER_CERT_PATH=~/.docker/zone1/
    $ docker --tlsverify ps

### 用`curl`连接安全的Docker端口

要使用`curl`测试API请求，你可以用下面三个额外的命令行参数：

    $ curl https://$HOST:2376/images/json \
      --cert ~/.docker/cert.pem \
      --key ~/.docker/key.pem \
      --cacert ~/.docker/ca.pem

## 相关信息

* [在镜像仓库里使用证书校验客户端](certificates.md)
* [使用受信任镜像](trust/index.md)
