---
description: Description of labels, which are used to manage metadata on Docker objects.
keywords: Usage, user guide, labels, metadata, docker, documentation, examples, annotating
title: Docker object labels
---

标签是用来将元数据应用到Docker对象中的一种机制，这样的Docker对象包括：

- 镜像
- 容器
- 本地Docker守护进程
- 数据卷
- 网络对象
- Swarm 节点
- Swarm 服务

你可以使用标签来组织你的镜像，记录你的许可信息，注解容器，数据卷，网络对象之间的关系，或者其他对你的应用或业务有价值的信息。

# 标签的键和值

一个标签是一个键值对，以字符串的形式保存。你可以给一个Docker对象指定多个标签，但是这个对象的每一个键值
对必须是唯一的。如果相同的键对应了多个不同的值，最近被赋值的内容会覆盖之前的内容。

## 键格式的建议

一个标签的 _键_ 是键值对左边的部分。键是由字符数字组成，可以包含句点"."和短折线"-"。大部分Docker用户
使用由其他组织构建出来的镜像，下面的指南帮助大家在Docker对象上避免使用相同键名的标签，特别是你计划把标签作为
自动化机制的一部分的时候。

- 第三方工具的作者应该在键名上加上各自翻转之后的域名前缀，例如`com.example.some-label`。

- 不要在没有获得域名所有者允许的情况下，在键名上使用其域名。

- `com.docker.*`, `io.docker.*` 和 `org.dockerproject.*` 是保留键名，作为Docker官方内部使用的

- 标签的键名应该以小写的字母开头，以及只应该包含小写的数字，字母，英文句点"."以及短折线"-"。
  连续的英文句点"."和短折线"-"是不被允许的。

- 英文句点"."用来分开名字空间的“字段”。没有名字空间的标签键名是保留给命令行用户使用的，允许命令行用户
  使用较短的字符串来给Docker对象打标签。

这些建议目前并没有被强制执行，可能会需要额外的指南来适应特殊的场景。

## 值的使用建议

标签的值可以是任何能表示为字符串的类型，包括但不限于JSON, XML, CSV或者YAML数据格式。唯一的要求是
这些值可以首先根据其类型和结构被序列化为字符串。举例而言，为了将一个JSON对象序列化为字符串，你需要使用
JavaScript的`JSON.stringify()`方法。

由于Docker不会反序列化标签的值，因此当你查询或者过滤一个标签值的时候，你不能将JSON或者XML文档当作一个
嵌套的结构来看待，除非你使用第三方的工具来实现这样的功能。

# 管理Docker对象上的标签

每种支持标签的对象都有添加和管理标签的机制，同时有针对该对象的使用标签的方法。下面这些链接教大家如何在
Docker部署环境中使用标签。

在镜像，容器，本地Docker守护进程，数据卷和网络的标签在其生命周期内是静态的。为了更新这些标签你须要重新
创建这些对象。Swarm节点和Swarm服务的标签可以被动态更新。

- 镜像和容器
  - [给镜像添加标签](../reference/builder.md#label)
  - [在运行时重写容器的标签](../reference/commandline/run.md#set-metadata-on-container--l---label---label-file)
  - [查看镜像或者容器的标签](../reference/commandline/inspect.md)
  - [通过标签来过滤镜像](../reference/commandline/inspect.md#filtering)
  - [通过标签来过滤容器](../reference/commandline/ps.md#filtering)

- 本地Docker守护进程
  - [在运行时给Docker守护进程增加标签](../reference/commandline/dockerd.md)
  - [查看Docker守护进程的标签](../reference/commandline/info.md)

- 数据卷
  - [给数据卷增加标签](../reference/commandline/volume_create.md)
  - [查看数据卷的标签](../reference/commandline/volume_inspect.md)
  - [通过标签过滤数据卷](../reference/commandline/volume_ls.md#filtering)

- 网络
  - [给网络增加标签](../reference/commandline/network_create.md)
  - [查看网络的标签](../reference/commandline/network_inspect.md)
  - [通过标签来过滤网络](../reference/commandline/network_ls.md#filtering)

- Swarm 节点
  - [给Swarm节点增加或者更新标签](../reference/commandline/node_update.md#add-label-metadata-to-a-node)
  - [查看Swarm节点的标签](../reference/commandline/node_inspect.md)
  - [通过标签来过滤Swarm节点](../reference/commandline/node_ls.md#filtering)

- Swarm服务
  - [给Swarm服务增加标签](../reference/commandline/service_create.md#set-metadata-on-a-service-l-label)
  - [更新Swarm服务的标签](../reference/commandline/service_update.md)
  - [查看Swarm服务的标签](../reference/commandline/service_inspect.md)
  - [通过标签来过滤Swarm服务](../reference/commandline/service_ls.md#filtering)