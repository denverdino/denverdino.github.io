---
description: Sec
keywords: seccomp, security, docker, documentation
title: Secure Engine
---

此部分文档讨论在Docker Engine里可以配置和使用的安全特性。

* 你可以配置Docker trust，让用户能够pull/push可信镜像。要了解如何做，可以查看[使用受信镜像](trust/index.md) 

* 你可以保护Docker daemon socket，确保只有可信Docker客户端能访问。更多信息参考[保护Docker daemon socket](https.md)

* 你可以使用基于证书的客户端服务器认证，以校验Docker daemon有权限访问registry上的镜像。详见[在仓库上使用证书校验客户端](certificates.md)


* 你可以配置安全计算模型(Seccomp)策略以加固容器内的系统调用。更多信息详见[针对Docker的Seccomp安全profiles](seccomp.md)

* 官方*.deb*包安装了Docker的AppArmor profile，关于此profile的更多信息以及修改profile，参见[针对Docker的AppArmor安全profiles](apparmor.md)

