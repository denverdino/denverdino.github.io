---
description: Learn how to use the Docker Hub to manage Docker images and work flow
keywords: repo, Docker Hub, Docker Hub, registry, index, repositories, usage, pull image, push image, image, documentation
redirect_from:
- /engine/userguide/containers/dockerrepos/
- /engine/userguide/dockerrepos/
title: 在Docker Hub上面存储镜像
---

到目前为止，您已经学会了如何使用命令行在本地主机上运行Docker。您学习了如何[拉取镜像](usingdocker.md)从现有的镜像构建容器，您已经学会了如何[创建自己的镜像](dockerimages.md)。

接下来，您将学习如何使用[Docker Hub](https://hub.docker.com)来简化和增强您的Docker工作流。

[Docker Hub](https://hub.docker.com)是由Docker公司维护的公共镜像仓库。它包含可以下载并用于构建容器的镜像。它还提供身份验证，工作组，工作流工具(如Webhooks和构建触发器)和隐私工具(如用于存储非共享镜像的私有仓库)。

## Docker命令和Docker Hub

Docker本身通过`docker search`，`pull`，`login`和`push`命令提供对Docker Hub服务的访问。此页面将显示这些命令的工作原理。

### 创建和登录帐户

在尝试Engine CLI命令之前，请创建Docker ID(如果您尚未创建Docker ID)。您可以通过[Docker Hub](https://hub.docker.com/)创建Docker ID。一旦您有一个Docker ID，从Engine CLI登录您的帐户：

```bash
$ docker login
```

`login`命令将您的Docker ID身份验证凭据存储在`$HOME/.docker/config.json`中(对于Bash shell而言)。对于Windows`cmd`用户，此文件的位置是`％HOME％\.docker\config.json`;对于PowerShell用户，位置是`$env:Home\.docker\config.json`。

一旦从命令行登录，您可以使用`commit`和`push` 等Engine子命令与Docker Hub上的仓库进行交互。

## 搜索镜像

您可以通过其搜索界面或使用命令行界面搜索[Docker Hub](https://hub.docker.com)镜像仓库。可以通过镜像名称，用户名或描述查找镜像：

	$ docker search centos
    NAME           DESCRIPTION                                     STARS     OFFICIAL   AUTOMATED
    centos         The official build of CentOS                    1223      [OK]
    tianon/centos  CentOS 5 and 6, created using rinse instea...   33
    ...
    
在那里您可以看到两个示例结果：`centos`和`tianon/centos`。第二个结果表明它来自一个名为`tianon`的用户的公共仓库，而第一个结果`centos`没有显式地列出一个仓库，这意味着它来自受信任的顶级命名空间[Official Repositories](/docker-hub/official_repos/). `/`字符将用户的存储库与映像名称分隔开。

找到所需的镜像后，您可以使用`docker pull <imagename>`下载：

	$ docker pull centos

    Using default tag: latest
    latest: Pulling from library/centos
    f1b10cd84249: Pull complete
    c852f6d61e65: Pull complete
    7322fbe74aa5: Pull complete
    Digest: sha256:90305c9112250c7e3746425477f1c4ef112b03b4abe78c612e092037bfecc3b7
    Status: Downloaded newer image for centos:latest

现在您有一个可以运行容器的镜像了。

### 指定版本或最新版本

使用`docker pull centos`相当于使用`docker pull centos:latest`。您可以通过更精确的指定镜像的版本来选择您需要的镜像。

例如，要拉取镜像`centos`的centos5版本，请使用`docker pull centos:centos5`。在这个例子中，`centos5`是一个标签，它标记`centos`仓库中一个指定镜像的版本。

要查找仓库的当前可用版本的标签列表，请参阅[Docker Hub](https://hub.docker.com)镜像仓库。

## 贡献给Docker Hub

任何人都可以从[Docker Hub](https://hub.docker.com)官方镜像仓库中提取公开的镜像，但是如果您想共享自己的镜像给其他人，那么您必须先[注册](/docker-hub/accounts)。

## 将镜像推送到Docker Hub

为了将镜像推送到其镜像仓库，您需要命名一个镜像或将容器提交到一个命名的镜像，参考[这儿](dockerimages.md)。

现在，您可以将此镜像推送到由其名称或标签指定的镜像仓库。

	$ docker push yourname/newimage

然后镜像将上传到镜像仓库，并供您的团队和/或社区使用。

## Docker Hub的特性

让我们仔细看看Docker Hub的一些功能。您可以在这里找到更多信息[这儿](/docker-hub/)。

* 私人存储库
* 组织和团队
* 自动构建
* Webhooks

### 私有镜像仓库

有时您有一些私有镜像，您不想公开分享给大家。因此，Docker Hub允许您拥有私有仓库。访问[这里](https://hub.docker.com/account/billing-plans/)。

### 组织和团队

私有仓库的一个有用的方面是，您只能与您的组织或团队的成员共享它们。 Docker Hub允许您创建组织，您可以与同事协作并管理私有仓库。您可以在这里了解如何[创建和管理组织](https://hub.docker.com/organizations/)。

### 自动构建

自动构建会直接在Docker Hub上自动化的从[GitHub](https://www.github.com)或[Bitbucket](http://bitbucket.com)构建和更新镜像。它的工作原理是通过向选定的GitHub或Bitbucket仓库添加一个钩子，在推送代码时触发Docker Hub上镜像的构建和更新。

#### 设置自动构建

1. 创建一个[Docker Hub帐户](https://hub.docker.com/)并登录。
2. 在["已关联的帐户和服务"](https://hub.docker.com/account/authorized-services/)页面上链接您的GitHub或Bitbucket帐户。
3. 从"创建"下拉菜单中选择"创建自动构建"
4. 选择一个包含`Dockerfile`的GitHub或Bitbucket项目。
5. 选择要构建的分支（默认是`master`分支）。
6. 给自动构建一个名称。
7. 为构建分配一个可选的Docker标签。
8. 指定`Dockerfile`所在的位置。默认是`/`。

一旦配置了自动构建，它将自动触发构建，并在几分钟后，您应该在[Docker Hub](https://hub.docker.com)镜像仓库上看到新的自动构建。它将保持与您的GitHub和Bitbucket仓库同步，直到您停用自动构建。

要检查自动构建存储库的输出和状态，请单击["Your Repositories"](https://registry.hub.docker.com/repos/)页面中的仓库名称。自动构建由仓库名称旁边的复选框图标标识。在镜像仓库详细信息页面中，您可以单击"Build Details"选项卡查看由Docker Hub触发的所有构建的状态和输出。

创建自动构建后，可以停用或删除它。但是不能使用`docker push`命令推送到自动构建的仓库。您只能通过将代码提交到GitHub或Bitbucket存储库来管理它。

您可以为每个仓库创建多个自动构建，并将其配置为指向特定的`Dockerfile`或Git分支。

#### 构建触发器

自动构建也可以通过Docker Hub上的URL触发。这允许您根据需要重新构建一个自动构建的镜像。

### Webhooks

Webhook被附加到您的镜像仓库上，当一个镜像或者已更新的镜像推送到镜像仓库时该Webhook将被触发。使用webhook，您可以指定在推送镜像时需要传递的目标URL和有效的JSON内容。

请参阅Docker Hub文档[有关webhooks的更多信息](/docker-hub/repos/#webhooks)

## 下一步

去使用Docker！