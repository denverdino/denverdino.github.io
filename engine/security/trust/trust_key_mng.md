---
description: Manage keys for content trust
keywords: trust, security, root,  keys, repository
title: Manage keys for content trust
---

一个镜像tag的信任管理是通过对不同秘钥的使用。Docker的内容信任使用5种类型的秘钥：

| 秘钥                 | 描述                                                                                                                                                                                                                                                                                                                                                                         |
|---------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 根秘钥         | 一个镜像tag的信任安全根。当信任安全开启，你需要一次性的创建根秘钥。也被称为离线秘钥，因为它被离线保存。|
| 目标          | 这个秘钥允许你签名镜像tag并管理代理包括代理秘钥或允许的代理路径。也被称作仓库秘钥。这个秘钥决定了哪些tag可以被签名到一个镜像仓库。 |
| 快照         | 这个秘钥签名了当前镜像tag的集合，防止最小和最大攻击。
| 时间戳        | 这个秘钥允许Docker镜像仓库有最新的安全防护，而不需要客户端周期性的刷新内容。 |
| 代理       | 代理秘钥是可选的tag秘钥。它允许你代理签名的镜像tag到其他发布者，而不需要分享你的目标秘钥。 |

当第一次使用信任安全`docker push`，根秘钥、目标秘钥、快照秘钥和时间戳秘钥会为这个镜像仓库自动生成。

- 根秘钥和目标秘钥被生成和存储在本地客户端。

- 时间戳秘钥和快照秘钥被安全生成和存储在签名服务端，它和Docker Registry部署在一块。这些秘钥被一个没有暴露到互联网的加密后台服务生成。

代理秘钥是可选的，没有在通常的docker工作流中生成。他们可以被
[手动创建并加到仓库](trust_delegation.md#generating-delegation-keys).

提示：在Docker Engine 1.11之前，快照秘钥也被生成和存储在客户端。 对于使用新版本创建的仓库，[使用 Notary 命令行在本地再次管理快照秘钥](/notary/advanced_usage.md#rotate-keys) .

## 选择一个口令

给根秘钥和仓库秘钥选择的口令应该随机创建并存储在密码管理器。拥有仓库秘钥使得用户可以对一个仓库内的镜像tag进行签名。
口令用来加密你的秘钥并且保证即使笔记本电脑丢失或者未预期的备份也不会导致私有秘钥内容有风险。

## 备份你的秘钥

所有的Docker信任秘钥都被你创建时的口令所加密。即使如此，你仍然应该小心你备份他们的位置。创建两个加密USB秘钥是好的实践。

在一个安全、加密的地点保存你的秘钥是非常重要的。丢失仓库秘钥是可恢复的。丢失根秘钥是不能恢复的。

Docker 客户端在目录`~/.docker/trust/private` 存储了这些秘钥。在备份他们之前，你应该把它们`tar`到一个档案。

```bash
$ umask 077; tar -zcvf private_keys_backup.tar.gz ~/.docker/trust/private; umask 022
```

## 硬件存储和加密

Docker内容信任可以从一个Yubikey 4设备存储和加密根秘钥。Yubikey比存储在文件系统更好。当你用内容安全初始化一个新的仓库，
Docker Engine 在本地查找根秘钥。如果没有找到并且yubikey 4存在，Docker Engine会在Yubikey 4创建一个新的根秘钥。
了解更多细节，请咨询[Notary 文档](/notary/advanced_usage.md#use-a-yubikey).

在Docker Engine 1.11之前, 这个特性只在体验分支可用。

## 秘钥丢失

如果一个发布者丢失了秘钥，意味着丢失了对你仓库进行加密信任内容的能力。
如果你丢失了秘钥，联系 [Docker Support](https://support.docker.com) (support@docker.com)去重置仓库状态。

这个丢失也需要每一个之前已经拉取相应镜像的消费者手动介入处理。镜像消费者会为已经下载的内容获得一个错误。

```
Warning: potential malicious behavior - trust data has insufficient signatures for remote repository docker.io/my/image: valid signatures did not meet threshold
```

要纠正这个，他们需要用新的秘钥重新下载镜像tag。


## 相关信息

* [Docker 内容信任](content_trust.md)
* [内容信任自动化](trust_automation.md)
* [内容信任-代理](trust_delegation.md)
* [在内容信任沙箱中运行](trust_sandbox.md)