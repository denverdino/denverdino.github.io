---
description: Delegations for content trust
keywords: trust, security, delegations, keys, repository
title: Delegations for content trust
---

Docker Engine 支持使用 `targets/releases` 代理作为一个信任的镜像tag的标准源。

使用这个代理秘钥允许你和其他发布者合作，而不需要分享你的仓库秘钥（你的目标密码和快照秘钥的联合，
请参见"[管理内容信任秘钥](trust_key_mng.md)" ).
一个协作者可以保存他们自己的代理秘钥私有。

`targets/releases` 代理目前是一个可选的特性 - 要使用代理，你必须使用Notary 命令行:

1. [下载客户端](https://github.com/docker/notary/releases) 并确保在你的路径可用。

2. 用以下内容创建配置文件 `~/.notary/config.json` :

	```
	{
	  "trust_dir" : "~/.docker/trust",
	  "remote_server": {
	    "url": "https://notary.docker.io"
	  }
	}
	```

	这个告诉Notary Docker内容安全数据存储在哪里, 并使用Docker Hub管理镜像使用的Notary server。

更多详情信息关于如何使用Notary，超出了默认Doceker内容安全的用例场景，请参看
[Notary 命令行文档](/notary/getting_started.md).

注意当发布和枚举代理使用Notary客户端时会有变化，Docker Hub的证书是需要的。

## 生成代理秘钥

你的协作者需要生产一个私钥（RSA或ECDSA），然后给你公钥，你可以把公钥加入到`targets/releases`代理。

最简单的方式是通过OpenSSL为他们创建秘钥.这里是一个如何通过生成2048位RSA（所有的RSA秘钥都至少2048位）密码的示例。

```
$ openssl genrsa -out delegation.key 2048
Generating RSA private key, 2048 bit long modulus
....................................................+++
............+++
e is 65537 (0x10001)

```

他们需要保持 `delegation.key` 私有 - 这就是用来签名tag的。

接下来他们需要生成一个包含公钥的x509证书，并把它给你。这里是生成CSR（证书签名请求）的命令：

```
$ openssl req -new -sha256 -key delegation.key -out delegation.csr
```

接下来他们可以把它发送给你用来签名证书的CA，或者他们自签名证书（这个示例里，创建一个有效期一年的证书）:

```
$ openssl x509 -req -days 365 -in delegation.csr -signkey delegation.key -out delegation.crt
```

接下来他们需要给你 `delegation.crt`,不管是自签名还是被CA签发。

## 给已有的仓库增加一个代理秘钥

如果你的仓库在Docker Engine 1.11之前创建，在增加任何代理前，你需要轮转服务端的快照秘钥，这样协助者不需要你的快照秘钥去签名和发布tag：

```
$ notary key rotate docker.io/<username>/<imagename> snapshot -r
```

这个会告诉Notary去轮转你特定镜像仓库的秘钥 - 注意你必须包含`docker.io/`前缀。
`snapshot -r`指明你希望轮转快照秘钥并且希望服务器去管理它(`-r`代表远程)。


当添加一个代理，你必须从你希望代理的协作者获得一个PEM编码的x509证书以及公钥(trust_delegation.md#generating-delegation-keys)。

假设你有`delegation.crt`的证书，你可以添加一个代理给这个用户，然后发布这个代理变化：

```
$ notary delegation add docker.io/<username>/<imagename> targets/releases delegation.crt --all-paths
$ notary publish docker.io/<username>/<imagename>
```

上述的示例展示了添加代理`targets/releases`到镜像仓库的请求过程，如果它不存在的话。
确保使用`targets/releases`- Notary支持多个代理角色，这样如果你搞错代理名称，Notary命令行也不会出错。
不管怎样，Docker Engine支持只从`targets/releases`读取。

它也将协助者的公钥加入到代理，使得一旦拥有相对于公钥的私钥，他们可以对`targets/releases` 代理进行签名。
发布这些变化可以告诉服务器关于对`targets/releases`代理的变化。
发布完，可以查看代理信息来保证你正确地添加了秘钥到`targets/releases`：

```
$ notary delegation list docker.io/<username>/<imagename>

      ROLE               PATHS                                   KEY IDS                                THRESHOLD
---------------------------------------------------------------------------------------------------------------
  targets/releases   "" <all paths>  729c7094a8210fd1e780e7b17b7bb55c9a28a48b871b07f65d97baf93898523a   1
```

你可以看到 `targets/releases` 和它的路径以及你刚才添加的秘钥ID。

Notary 目前没有映射协助者的名称到秘钥，所以我们推荐你一次一个地增加和罗列代理秘钥，并且自己维护一个秘钥ID和协助者的映射关系，便于你需要移除一个协作者。


## 从存在的仓库移除一个代理秘钥

要收回一个协助者对你镜像仓库签名的权限，你必须知道他们秘钥的ID，因为你需要从`targets/releases`代理移除他们的秘钥。

```
$ notary delegation remove docker.io/<username>/<imagename> targets/releases 729c7094a8210fd1e780e7b17b7bb55c9a28a48b871b07f65d97baf93898523a

Removal of delegation role targets/releases with keys [729c7094a8210fd1e780e7b17b7bb55c9a28a48b871b07f65d97baf93898523a], to repository "docker.io/<username>/<imagename>" staged for next publish.
```

一旦你发布，权限收回会生效:

```
$ notary publish docker.io/<username>/<imagename>
```

注意从`targets/releases`代理通过移除所有的秘钥，代理（和已经签名的tag）会被清除。
这意味着这些tag都会被删除，而最终留下之前被目标秘钥直接签名的旧的、遗留的tag。

## 从仓库完整移除`targets/releases`代理

如果你已经决定这些代理没用，你可以完整删除`targets/releases`代理。这个也会移除所有当前在`targets/releases`中的tag，
不管怎样，最终留下之前被目标秘钥直接签名的旧的、遗留的tag。

删除`targets/releases` 代理:

```
$ notary delegation remove docker.io/<username>/<imagename> targets/releases

Are you sure you want to remove all data for this delegation? (yes/no)
yes

Forced removal (including all keys and paths) of delegation role targets/releases to repository "docker.io/<username>/<imagename>" staged for next publish.

$ notary publish docker.io/<username>/<imagename>
```

## 作为协助者上传信任数据

作为协作者，拥有已经加入到仓库`targets/releases`代理的私钥，你需要导入这些你生成到内容信任的私钥。

去做这件事，你可以运行:

```
$ notary key import delegation.key --role user
```

`delegation.key` 包含了你的PEM编码的私钥。

当你运行完命令，对任何包含`targets/releases`代理秘钥的仓库执行`docker push`，将会自动用导入的秘钥进行签名。

## `docker push` 行为

当用Docker内容安全运行`docker push`, 
如果`targets/releases`代理存在时，Docker Engine将会尝试签名并推送`targets/releases`代理。
如果代理不存在，并且目标秘钥存在的话，目标秘钥会被用来加签tag。

## `docker pull` 和 `docker build` 行为

当用Docker内容安全运行 `docker pull` or `docker build` 时, Docker
Engine 只会拉取通过`targets/releases` 代理角色签名的tag或者老的直接被目标秘钥签名的tag。

## 相关信息

* [Docker 内容信任](content_trust.md)
* [管理内容信任秘钥](trust_key_mng.md)
* [内容信任自动化](trust_automation.md)
* [在内容信任沙箱中运行](trust_sandbox.md)