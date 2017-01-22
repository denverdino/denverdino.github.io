---
description: Play in a trust sandbox
keywords: trust, security, root,  keys, repository, sandbox
title: Play in a content trust sandbox
---

本页解释了如何安装和使用一个沙箱来体验信任。
沙箱允许你在本地配置和尝试信任操作，而不影响你的生产环境镜像。

在操作沙箱之前，你应该已经阅读了 [Docker 内容信任](content_trust.md).

### 前提条件

这些操作引导假设你在Linux或macOS上运行。你可以在本机或虚拟机上运行这个沙箱。
你将需要提供权限去运行Docker命令在你的本机或虚拟机。

这个沙箱需要你安装两个Docker工具： Docker Engine >= 1.10.0和 Docker Compose >= 1.6.0。
要安装Docker Engine，从这里选择[支持平台列表](../../installation/index.md). 
要安装Docker Compose, 参见[详细操作](/compose/install/).

最后，你将需要在你本机或虚拟机上安装一个文本编辑器。

## 什么是沙箱?

如果你只是开箱即用信任，你只需要用Docker Engine客户端并访问Docker Hub。沙箱最小化一个产品信任环境，并且安装这些额外的组件。


| Container       | Description                                                                                                                                 |
|-----------------|---------------------------------------------------------------------------------------------------------------------------------------------|
| trustsandbox    | A container with the latest version of Docker Engine and with some preconfigured certificates. This is your sandbox where you can use the `docker` client to test trust operations. |
| Registry server | A local registry service.                                                                                                                 |
| Notary server   | The service that does all the heavy-lifting of managing trust                                                                               |

这意味着你将运行你自己的内容信任（Notary）服务和registry。如果你只是使用Docker Hub，你不需要这些组件。
这些已经内置在Docker Hub。对于沙箱而言，无论如何，你构建了你自己完整的模拟产品环境。

通过`trustsandbox`容器，你和本地registry而不是Docker Hub交互。
这意味着你每天使用的镜像仓库没有被使用。他们被保护起来。

当你在沙箱里测试，你也将创建根秘钥和仓库秘钥。
这个沙箱被配置为存储`trustsandbox`容器里的所有的秘钥和文件。
因为在沙箱里创建的这些秘钥只是用来测试，消耗了容器也会销毁他们。

通过使用docker-in-docker镜像来运行`trustsandbox`容器，你将不会污染你的真正docker daemon缓存（这些缓存保存了你之前上传下载的镜像）。
这些镜像会被存储在容器的一个匿名卷，可以在销毁容器后销毁。

## 构建沙箱

这个部分，你将使用Docker Compose去指定如何安装和连接`trustsandbox`容器Notary server和、Registry server。


1. 创建一个新的 `trustsandbox` 目录并切换到该目录。

        $ mkdir trustsandbox
        $ cd trustsandbox

2. 用你喜欢的编辑器创建文件`docker-compose.yml`。例如使用vim:

        $ touch docker-compose.yml
        $ vim docker-compose.yml

3. 在这个新文件里添加以下内容：

        version: "2"
        services:
          notaryserver:
            image: dockersecurity/notary_autobuilds:server-v0.4.2
            volumes:
              - notarycerts:/go/src/github.com/docker/notary/fixtures
            networks:
              - sandbox
            environment:
              - NOTARY_SERVER_STORAGE_TYPE=memory
              - NOTARY_SERVER_TRUST_SERVICE_TYPE=local
          sandboxregistry:
            image: registry:2.4.1
            networks:
              - sandbox
            container_name: sandboxregistry
          trustsandbox:
            image: docker:dind
            networks:
              - sandbox
            volumes:
              - notarycerts:/notarycerts
            privileged: true
            container_name: trustsandbox
            entrypoint: ""
            command: |-
                sh -c '
                    cp /notarycerts/root-ca.crt /usr/local/share/ca-certificates/root-ca.crt &&
                    update-ca-certificates &&
                    dockerd-entrypoint.sh --insecure-registry sandboxregistry:5000'
        volumes:
          notarycerts:
            external: false
        networks:
          sandbox:
            external: false

4. 保存并关闭文件.

5. 在本地运行容器

        $ docker-compose up -d

    当你第一次运行，docker-in-docker, Notary server, 和 registry的镜像会从Docker Hub下载。
    


## 在沙箱里测试

当一切准备好，你可以进入`trustsandbox`容器并开始测试Docker内容安全。
从你的宿主机，打开`trustsandbox`容器的shell.


    $ docker exec -it trustsandbox sh
    / #

### 测试一些信任操作

现在，你可以在`trustsandbox`容器里拉取一些镜像。

1.下载一个 `docker` 镜像去测试.

        / # docker pull docker/trusttest
        docker pull docker/trusttest
        Using default tag: latest
        latest: Pulling from docker/trusttest

        b3dbab3810fc: Pull complete
        a9539b34a6ab: Pull complete
        Digest: sha256:d149ab53f8718e987c3a3024bb8aa0e2caadf6c0328f1d9d850b2a2a67f2819a
        Status: Downloaded newer image for docker/trusttest:latest

2. 对它打一个tag，标记为推送到沙箱 registry:

        / # docker tag docker/trusttest sandboxregistry:5000/test/trusttest:latest

3. 开启内容信任

        / # export DOCKER_CONTENT_TRUST=1

4. 识别信任服务器

        / # export DOCKER_CONTENT_TRUST_SERVER=https://notaryserver:4443

    This step is only necessary because the sandbox is using its own server.
    Normally, if you are using the Docker Public Hub this step isn't necessary.

5. 拉取测试镜像.

        / # docker pull sandboxregistry:5000/test/trusttest
        Using default tag: latest
        Error: remote trust data does not exist for sandboxregistry:5000/test/trusttest: notaryserver:4443 does not have trust data for sandboxregistry:5000/test/trusttest

      你会看到一个错误，因为`notaryserver`上还没有内容。

6. 推送并加签这个信任镜像。

        / # docker push sandboxregistry:5000/test/trusttest:latest
        The push refers to a repository [sandboxregistry:5000/test/trusttest]
        5f70bf18a086: Pushed
        c22f7bc058a9: Pushed
        latest: digest: sha256:ebf59c538accdf160ef435f1a19938ab8c0d6bd96aef8d4ddd1b379edf15a926 size: 734
        Signing and pushing trust metadata
        You are about to create a new root signing key passphrase. This passphrase
        will be used to protect the most sensitive key in your signing system. Please
        choose a long, complex passphrase and be careful to keep the password and the
        key file itself secure and backed up. It is highly recommended that you use a
        password manager to generate the passphrase and keep it safe. There will be no
        way to recover this key. You can find the key in your config directory.
        Enter passphrase for new root key with ID 27ec255:
        Repeat passphrase for new root key with ID 27ec255:
        Enter passphrase for new repository key with ID 58233f9 (sandboxregistry:5000/test/trusttest):
        Repeat passphrase for new repository key with ID 58233f9 (sandboxregistry:5000/test/trusttest):
        Finished initializing "sandboxregistry:5000/test/trusttest"
        Successfully signed "sandboxregistry:5000/test/trusttest":latest

    因为你第一次推送到这个仓库，docker创建了新的根秘钥和仓库秘钥，并且向你要加密口令。如果你第二次推送，它只会向你要仓库口令去解密秘钥并再次加签。

7. 尝试拉取你刚才推送的镜像:

        / # docker pull sandboxregistry:5000/test/trusttest
        Using default tag: latest
        Pull (1 of 1): sandboxregistry:5000/test/trusttest:latest@sha256:ebf59c538accdf160ef435f1a19938ab8c0d6bd96aef8d4ddd1b379edf15a926
        sha256:ebf59c538accdf160ef435f1a19938ab8c0d6bd96aef8d4ddd1b379edf15a926: Pulling from test/trusttest
        Digest: sha256:ebf59c538accdf160ef435f1a19938ab8c0d6bd96aef8d4ddd1b379edf15a926
        Status: Downloaded newer image for sandboxregistry:5000/test/trusttest@sha256:ebf59c538accdf160ef435f1a19938ab8c0d6bd96aef8d4ddd1b379edf15a926
        Tagging sandboxregistry:5000/test/trusttest@sha256:ebf59c538accdf160ef435f1a19938ab8c0d6bd96aef8d4ddd1b379edf15a926 as sandboxregistry:5000/test/trusttest:latest


### 用恶意镜像测试

当信任内容开启，如果你尝试拉取数据损坏的镜像会发生什么？这个部分，你可以进入`sandboxregistry`，
并篡改一些数据。然后你尝试拉取它。

1. 离开`trustsandbox` shell并保持容器运行.

2. 在你的机器上开启一个新的交互终端，重新在`sandboxregistry`容器里打开一个shell。

        $ docker exec -it sandboxregistry bash
        root@65084fc6f047:/#

3. 列出`test/trusttest` 镜像的层信息:

        root@65084fc6f047:/# ls -l /var/lib/registry/docker/registry/v2/repositories/test/trusttest/_layers/sha256
        total 12
        drwxr-xr-x 2 root root 4096 Jun 10 17:26 a3ed95caeb02ffe68cdd9fd84406680ae93d633cb16422d00e8a7c22955b46d4
        drwxr-xr-x 2 root root 4096 Jun 10 17:26 aac0c133338db2b18ff054943cee3267fe50c75cdee969aed88b1992539ed042
        drwxr-xr-x 2 root root 4096 Jun 10 17:26 cc7629d1331a7362b5e5126beb5bf15ca0bf67eb41eab994c719a45de53255cd

4. 切换到registry存储目录中该镜像的其中一层（注意这是一个不同的目录）

        root@65084fc6f047:/# cd /var/lib/registry/docker/registry/v2/blobs/sha256/aa/aac0c133338db2b18ff054943cee3267fe50c75cdee969aed88b1992539ed042

5. 增加恶意数据到其中一个信任层:

        root@65084fc6f047:/# echo "Malicious data" > data

6. 回到你的`trustsandbox` 终端.

7. 列出信任镜像

        / # docker images | grep trusttest
        REPOSITORY                            TAG                 IMAGE ID            CREATED             SIZE
        docker/trusttest                      latest              cc7629d1331a        11 months ago       5.025 MB
        sandboxregistry:5000/test/trusttest   latest              cc7629d1331a        11 months ago       5.025 MB
        sandboxregistry:5000/test/trusttest   <none>              cc7629d1331a        11 months ago       5.025 MB

8. 从本地缓存中删除 `trusttest:latest` 镜像。

        / # docker rmi -f cc7629d1331a
        Untagged: docker/trusttest:latest
        Untagged: sandboxregistry:5000/test/trusttest:latest
        Untagged: sandboxregistry:5000/test/trusttest@sha256:ebf59c538accdf160ef435f1a19938ab8c0d6bd96aef8d4ddd1b379edf15a926
        Deleted: sha256:cc7629d1331a7362b5e5126beb5bf15ca0bf67eb41eab994c719a45de53255cd
        Deleted: sha256:2a1f6535dc6816ffadcdbe20590045e6cbf048d63fd4cc753a684c9bc01abeea
        Deleted: sha256:c22f7bc058a9a8ffeb32989b5d3338787e73855bf224af7aa162823da015d44c

    Docker不会重新下载已经缓存的镜像，但是我们想让Docker下载这个篡改的镜像并且因为它不正确而拒绝。

8. 重新下载镜像。这次会从registry下载，因为没有缓存。

        / # docker pull sandboxregistry:5000/test/trusttest
        Using default tag: latest
        Pull (1 of 1): sandboxregistry:5000/test/trusttest:latest@sha256:35d5bc26fd358da8320c137784fe590d8fcf9417263ef261653e8e1c7f15672e
        sha256:35d5bc26fd358da8320c137784fe590d8fcf9417263ef261653e8e1c7f15672e: Pulling from test/trusttest

        aac0c133338d: Retrying in 5 seconds
        a3ed95caeb02: Download complete
        error pulling image configuration: unexpected EOF

      你将发现下载没有结束，因为信任系统不能校验镜像。

## 更多沙箱尝试

现在，你在本机有了一个完整的Docker信任安全沙箱，可以随意玩耍并看看有什么会发生。
如果你发现了任何安全问题，请随时给我们发邮件<security@docker.com>。


## 清除沙箱

当你测试结束，希望清理所有你启动的服务和创建的匿名卷，只要在创建Docker Compose文件目录运行下面的命令：

        $ docker-compose down -v
