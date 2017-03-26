---
description: Docker 入门
keywords: beginner, getting started, Docker
redirect_from:
- /mac/step_two/
- /windows/step_two/
- /linux/step_two/
title: 什么是镜像和容器
---

Docker Engine为我们提供了能够支持 *镜像* 和 *容器* 的核心技术。
在上一节你或许已经运行了`docker run hello-world`，这条命令的执行可以被分为三个部分。

![Container Explainer](tutimg/container_explainer.png)

 *镜像* 是由一个文件系统和一些运行时需要的参数组成的。镜像没有状态，也不会再发生变化。
 而 *容器* 则是一个运行中的镜像实例。当你执行上面的命令时，Docker Engine是这样处理的：

* 检查你是否已经拥有 `hello-world` 这个软件镜像
* 从Docker Hub下载镜像
* 装载镜像到容器中并运行

一个镜像可能只运行一条简单的命令然后就退出（这也就是
`hello-world` 这个镜像所做的），这取决于它是如何构建的。

然而一个Docker镜像可以胜任更多的事情。例如一个镜像可以启动数据库这样复杂的软件，等待你
（或者其他人）来添加、存储数据，然后再等待下一个用户的操作。

那么是谁构建了 `hello-world` 这个镜像呢？在这个例子中是Docker，然而任何人都可以构建
镜像。Docker Engine允许人们（或公司）创建并且分享Docker镜像。通过Docker Engine，
你不必担心你的计算机是否能运行镜像中的软件，Docker容器 *can always run it*.

## 下一步

看，学习Docker很快吧？现在，你已经做好准备去用Docker做一些真正有趣的事情了。继续下一个
部分[找到并运行whalesay镜像](step_three.md)
