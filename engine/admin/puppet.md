---
description: Installing and using Puppet
keywords: puppet, installation, usage, docker,  documentation
redirect_from:
- /engine/articles/puppet/
title: Use Puppet
---

> *注意:* 请注意这是由社区提供安装方式。
> 唯一官方安装方式见
> [*Ubuntu*](../installation/linux/ubuntulinux.md)
> 。这个版本有些时候可能是过时的。

## 要求

在使用本文档前，你需要确保已经从 [Puppet Labs](https://puppetlabs.com) 安装了一个可以使用的Puppet。

由于这个模块目前使用官方 PPA 方式，因此只能在Ubuntu系统上运行。

## 安装
这个模块可以在这里找到 [Puppet
Forge](https://forge.puppetlabs.com/garethr/docker/) ，使用下面的命令安装本模块。

    $ puppet module install garethr/docker

如果你想下载源码，可以在这里找到
[GitHub](https://github.com/garethr/garethr-docker) 。

## 使用

这个模块提供一个puppet class 安装 Docker， 且有两个 defined types 去管理镜像和容器。

### 安装

    include 'docker'

### 镜像

下一步就是安装Docker 镜像。我们有一个 defined type 可以按如下方式使用:

    docker::image { 'ubuntu': }

这个等价于运行:

    $ docker pull ubuntu

注意一点，它只会在镜像名不存在的时候才会去下载。这会去下载一个很大的文件，因此在第一次运行的时候会耗费一些时间。由于这个原因， 这个 define 关闭了 exec type 的默认5分钟超时。当然，你可以使用下面的命令删除那些不需要的镜像:

    docker::image { 'ubuntu':
      ensure => 'absent',
    }

### 容器

现在，你拥有了一个镜像，你可以去运行命令在Docker 管理的容器内。

    docker::run { 'helloworld':
      image   => 'ubuntu',
      command => '/bin/sh -c "while true; do echo hello world; sleep 1; done"',
    }

这个等价于运行下面的命令， 但是是在upstart后: 

    $ docker run -d ubuntu /bin/sh -c "while true; do echo hello world; sleep 1; done"

剩余的容器运行时可选项如下:

    docker::run { 'helloworld':
      image        => 'ubuntu',
      command      => '/bin/sh -c "while true; do echo hello world; sleep 1; done"',
      ports        => ['4444', '4555'],
      volumes      => ['/var/lib/couchdb', '/var/log'],
      volumes_from => '6446ea52fbc9',
      memory_limit => 10485760, # bytes
      username     => 'example',
      hostname     => 'example.com',
      env          => ['FOO=BAR', 'FOO2=BAR2'],
      dns          => ['8.8.8.8', '8.8.4.4'],
    }

> *注意:*
> `ports`, `env`, `dns` 和 `volumes` 属性可以设置为单独的一个字符串 
> 或者一个字符串数组。