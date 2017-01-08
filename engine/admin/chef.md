---
description: Installation and using Docker via Chef
keywords: chef, installation, usage, docker,  documentation
redirect_from:
- /engine/articles/chef/
title: Use Chef
---

> **注意**:
> 这是由社区贡献的

## 前提条件

请确保您已经安装了[Chef](https://www.chef.io/). 这个cookbook支持多种操作系统。

## 安装

可以在[Chef Supermarket](https://supermarket.chef.io/cookbooks/docker) 搜索到docker cookbook，并且通过你喜欢或者擅长的cookbook依赖工具下载。

源代码可以在
[GitHub](https://github.com/someara/chef-docker)找到.

使用方法
-----
- 在您cookbook的metadata.rb文件里添加```depends 'docker', '~> 2.0'```代码
- 像使用其他Chef模块(file, template, directory, package等等)一样使用docker cookbook里的模块


```ruby
docker_service 'default' do
  action [:create, :start]
end

docker_image 'busybox' do
  action :pull
end

docker_container 'an echo server' do
  repo 'busybox'
  port '1234:1234'
  command "nc -ll -p 1234 -e /bin/cat"
end
```

## 入门
以下的例子是如何用Chef下载最新版本的`nginx`容器镜像并且运行容器。

```ruby
# 下载最新的容器镜像
docker_image 'nginx' do
  tag 'latest'
  action :pull
end

# 暴露`80`端口，并运行容器
docker_container 'my_nginx' do
  repo 'nginx'
  tag 'latest'
  port '80:80'
  binds [ '/some/local/files/:/etc/nginx/conf.d' ]
  host_name 'www'
  domain_name 'computers.biz'
  env 'FOO=bar'
  subscribes :redeploy, 'docker_image[nginx]'
end
```
