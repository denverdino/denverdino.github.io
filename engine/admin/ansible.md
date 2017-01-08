---
description: Installation and using Docker via Ansible
keywords: ansible, installation, usage, docker,  documentation
title: Use Ansible
---

> **注意**:
> 这是由社区贡献的

## 前提条件

请确保您使用的是[Ansible](https://www.ansible.com/) 2.1.0 或者以上版本。


需要使用docker的module的管理对象节点需要满足如下前提:

```
python >= 2.6
docker-py >= 1.7.0
Docker API >= 1.20
```

## 安装

`docker_container`模块是内置在Ansible中的核心模块。

## 使用方法

以下的Task例子是下载最新版本的`nginx`容器镜像并且运行容器。如何在这里定义网络地址和端口请参阅[变量](https://docs.ansible.com/ansible/playbooks_variables.html).

```
---
- name: nginx container
  docker:
    name: nginx
    image: nginx
    state: reloaded
    ports:
    - "{{ nginx_bind_address }}:{{ nginx_port }}:{{ nginx_port }}"
    cap_drop: all
    cap_add:
      - setgid
      - setuid
    pull: always
    restart_policy: on-failure
    restart_policy_retry: 3
    volumes:
      - /some/nginx.conf:/etc/nginx/nginx.conf:ro
  tags:
    - docker_container
    - nginx
...
```

## 文档

关于`ansible_container` 模块的文档，请参阅
[docs.ansible.com](https://docs.ansible.com/ansible/docker_container_module.html).

关于容器镜像，网络和服务的文档，请参阅[docs.ansible.com](https://docs.ansible.com/ansible/list_of_cloud_modules.html#docker).