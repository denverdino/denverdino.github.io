---
description: Docker 入门
keywords: beginner, getting started, Docker
redirect_from:
- /mac/step_three/
- /windows/step_three/
- /linux/step_three/
title: 找到并运行whalesay镜像
---

世界各地的人们都在创建Docker镜像。你可以通过浏览Docker Hub来找到这些镜像。在这一小节中，
你将搜索并找到一个镜像，在接下来的入门章节中你都会需要用到它。

## Step 1: 找到 whalesay 镜像

1. 打开网页浏览器，浏览[Docker Hub](https://hub.docker.com/?utm_source=getting_started_guide&utm_medium=embedded_MacOSX&utm_campaign=find_whalesay)
    ![Browse Docker Hub](tutimg/browse_and_search.png)
   Docker Hub 中包含了来自个人（比如你）或者机构（比如 RedHat，IBM, Google等）的镜像。

2. 在搜索框中输入 `whalesay`.

    ![Browse Docker Hub](tutimg/image_found.png)

3. 点击搜索结果中的 **docker/whalesay** 镜像。

    浏览器中展示了 **whalesay** 镜像的仓库。

    ![Browse Docker Hub](tutimg/whale_repo.png)

    每个镜像仓库中都包含了关于这个镜像的一些信息，这些信息包括这个镜像中有哪些软件、
    怎样使用它等等。你可能注意到 **whalesay** 镜像是建立在名为 Ubuntu 的 Linux 发行版
    之上的。在下一步中，你可以在自己的机器上运行 **whalesay** 镜像。

## Step 2: 运行 whalesay 镜像

首先确保 Docker 正在运行中。在 Docker for Mac 和 Docker for Windows 中，这意味着
Docker 的鲸鱼图标正显示在状态栏上。

1. 打开一个命令行终端。

2. 输入 `docker run docker/whalesay cowsay boo` 命令并回车

    这个命令将在一个容器中运行 **whalesay** 镜像。你的终端输出看起来应该像下面这样：

        $ docker run docker/whalesay cowsay boo
        Unable to find image 'docker/whalesay:latest' locally
        latest: Pulling from docker/whalesay
        e9e06b06e14c: Pull complete
        a82efea989f9: Pull complete
        37bea4ee0c81: Pull complete
        07f8e8c5e660: Pull complete
        676c4a1897e6: Pull complete
        5b74edbcaa5b: Pull complete
        1722f41ddcb5: Pull complete
        99da72cfe067: Pull complete
        5d5bd9951e26: Pull complete
        fb434121fc77: Already exists
        Digest: sha256:d6ee73f978a366cf97974115abe9c4099ed59c6f75c23d03c64446bb9cd49163
        Status: Downloaded newer image for docker/whalesay:latest
         _____
        < boo >
         -----
            \
             \
              \     
                            ##        .            
                      ## ## ##       ==            
                   ## ## ## ##      ===            
               /""""""""""""""""___/ ===        
          ~~~ {~~ ~~~~ ~~~ ~~~~ ~~ ~ /  ===- ~~~   
               \______ o          __/            
                \    \        __/             
                  \____\______/   

  当第一次运行软件镜像时，`docker` 命令在本地系统中寻找它，如果本地没有这个镜像，
  `docker` 命令将从 Docker Hub 中获取它。

3. 我们再从命令行中输入 `docker images` 并回车

    这个命令会列出在本地系统中的所有镜像，你应该在里面看到 `docker/whalesay` 这个镜像。

        $ docker images
        REPOSITORY           TAG         IMAGE ID            CREATED            SIZE
        docker/whalesay      latest      fb434121fc77        3 hours ago        247 MB
        hello-world          latest      91c95931e552        5 weeks ago        910 B

    当你在容器中运行一个镜像时，Docker 将镜像下载至你的计算机。这个镜像的本地副本能够
    节约你的时间。Docker 只会在 Hub 上该镜像发生变化时才会再次下载它。当然，你也可以自己
    把本地系统中的镜像删除掉，再次运行时 Docker 会重新下载。接下来你将会学习到更多关于
    这方面的内容。 我们现在先把这个镜像放在一边，等会儿会用到它。

4. 花点时间来看看 **whalesay** 容器

    试试用一个单词或短语来运行 `whalesay` 镜像。

        $ docker run docker/whalesay cowsay boo-boo
         _________
        < boo-boo >
         ---------
            \
             \
              \     
                            ##        .            
                      ## ## ##       ==            
                   ## ## ## ##      ===            
               /""""""""""""""""___/ ===        
          ~~~ {~~ ~~~~ ~~~ ~~~~ ~~ ~ /  ===- ~~~   
               \______ o          __/            
                \    \        __/             
                  \____\______/   

## 下一步

在这个页面中，你学习到了如何在 Docker Hub 上搜索镜像，并且通过命令行来运行了一个镜像。
想想看，你可以非常容易地在自己的 Mac 上运行一段 Linux 程序。
你学习到了在自己的计算机上运行镜像的副本。现在，你已经做好准备来创建自己的 Docker 镜像了。
前行下一部分[构建自己的镜像](step_four.md)

&nbsp;
