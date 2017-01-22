---
description: How PKI works in swarm mode
keywords:
- docker
- container
- cluster
- swarm mode
- node
- tls
- pki
title: How PKI works in swarm mode
---

Docker引擎中内置的公钥基础架构（PKI）系统可以简化安全部署容器编排系统的过程。群中的节点使用传输层安全（TLS）来认证、授权和加密它们与群中的其他节点之间的通信。

当你运行`docker swarm init`创建一个swarm时，Docker引擎将自己指定为一个管理节点。默认情况下，管理节点为自己生成一个新的根证书（CA）以及用于保证其他节点相互通信的密钥对。如果你喜欢，你可以通过`--external-ca`标志来指定一个导入的根证书。参考[Docker Swarm初始化](../../ reference / commandline / swarm_init.md) CLI。

当您将其他节点加入swarm时，管理节点还会生成两个令牌：一个工作节点令牌和一个管理节点令牌。每个令牌包括根证书的摘要和随机生成的秘钥。当节点加入群集时，它使用摘要来验证远端机器的的根证书。它使用秘钥来确保节点是合法的节点。

每次新节点加入群集时，管理节点向包含随机节点ID的节点颁发证书，以标识证书公用名（CN）下的节点和组织单位（OU）下的角色。节点ID用作集群中节点生存期的安全身份。

下图说明了工作管理节点和工作节点如何使用TLS 1.2进行加密通信。

```bash
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            3b:1c:06:91:73:fb:16:ff:69:c3:f7:a2:fe:96:c1:73:e2:80:97:3b
        Signature Algorithm: ecdsa-with-SHA256
        Issuer: CN=swarm-ca
        Validity
            Not Before: Aug 30 02:39:00 2016 GMT
            Not After : Nov 28 03:39:00 2016 GMT
        Subject: O=ec2adilxf4ngv7ev8fwsi61i7, OU=swarm-worker, CN=dw02poa4vqvzxi5c10gm4pq2g
...snip...
```

默认情况下，swarm中的每个节点每三个月更新一次证书。您可以运行`docker swarm update --cert-expiry <TIME PERIOD>`来配置节点更新证书的频率。最小值为1小时。参考[docker swarm更新](../../ reference / commandline / swarm_update.md) CLI参考。


## 更多

* 参考 [节点](nodes.md) 如何工作.
* 参考 Swarm模式下[服务](services.md) 如何工作。
