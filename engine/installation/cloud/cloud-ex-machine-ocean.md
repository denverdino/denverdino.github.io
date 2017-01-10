---
description: Example of using Docker Machine to install Docker Engine on a cloud provider, using Digital Ocean.
keywords: cloud, docker, machine, documentation, installation, digitalocean
title: 'Example: Use Docker Machine to provision cloud hosts'
---

你需要安装并运行Docker Machine，创建一个云服务账号

然后提供账号验证，安全验证及更多配置信息作为 `docker-machine create` 命令参数，不同云服务商驱动的参数标识是不同的。例如Digital Ocean访问令牌对应参数为 `--digitalocean-access-token`

下面的实例我们将展示如何创建一台安装Docker环境的 <a href="https://digitalocean.com" target="_blank">Digital Ocean</a> _Droplet_ (云服务器).


### 步骤 1. 创建 Digital Ocean 账号并登陆

如果你还没有账号, 请到 <a href="https://digitalocean.com" target="_blank">Digital Ocean</a>, 创建账号, 然后登录.

### 步骤 2. 创建个人访问令牌

按下面的步骤创建你的访问令牌:

1.  前往 Digital Ocean 管理控制台，点击顶部的 **API** .

    ![Click API in Digital Ocean console](../images/ocean_click_api.png)

2. 点击 **Generate New Token** 生产令牌.

   ![Generate token](../images/ocean_gen_token.png)

3.  给令牌起一个便于记忆的名称 (如. "machine"), 确定要选中 **Write (Optional)** , 点击 **Generate Token**.

    ![Name and generate token](../images/ocean_token_create.png)

4.  获取 (复制到粘贴板) 生成的长字符串并将它保存到安全的位置.

    ![Copy and save personal access token](../images/ocean_save_token.png)

    这个访问令牌在后面创建云服务器时会使用.

### 步骤 3. 安装 Docker Machine

1. 如果你还没有安装Docker Machine, 在你机器上安装.

  * <a href="https://docs.docker.com/engine/installation/mac/" target="_blank"> 如何在macOS上安装Docker Machine</a>

  * <a href="https://docs.docker.com/engine/installation/windows/" target="_blank">如何在Windows上安装Docker Machine</a>

  * <a href="https://docs.docker.com/machine/install-machine/" target="_blank">安装Docker Machine</a> (Linux版本)

2. 在终端使用 `docker-machine ls` 获取 Docker Machine 列表和他们的状态

    ```
    $ docker-machine ls
    NAME      ACTIVE   DRIVER       STATE     URL                         SWARM
    default   *        virtualbox   Running   tcp:////xxx.xxx.xx.xxx:xxxx
    ```

3.  运行一些Docker命令，验证Docker Engine是否已经运行.

    我们仍将使用 `docker run hello-world` , 你可试试这些命令 `docker ps`,  `docker run docker/whalesay cowsay boo`, 或是其他命令来验证Docker是否运行.

    ```
    $ docker run hello-world

    Hello from Docker.
    This message shows that your installation appears to be working correctly.
    ...
    ```

### 步骤 4. 使用 Machine 来创建Droplet

1.  执行 `docker-machine create` ，使用 `digitalocean` 驱动，传入你的令牌参数 `--digitalocean-access-token` , 你也可以设置云服务器名称.

    例如, 我们创建一台Droplet，命名为 "docker-sandbox".

    ```
    $ docker-machine create --driver digitalocean --digitalocean-access-token xxxxx docker-sandbox
    Running pre-create checks...
    Creating machine...
    (docker-sandbox) OUT | Creating SSH key...
    (docker-sandbox) OUT | Creating Digital Ocean droplet...
    (docker-sandbox) OUT | Waiting for IP address to be assigned to the Droplet...
    Waiting for machine to be running, this may take a few minutes...
    Machine is running, waiting for SSH to be available...
    Detecting operating system of created instance...
    Detecting the provisioner...
    Provisioning created instance...
    Copying certs to the local machine directory...
    Copying certs to the remote machine...
    Setting Docker configuration on the remote daemon...
    To see how to connect Docker to this machine, run: docker-machine env docker-sandbox
    ```

      当Droplet创建完成后, Docker会生成一个唯一的SSH key并将它保存在本机`~/.docker/machines`下. 最初使用它来提供主机，后来可以使用它通过 `docker-machine ssh` 命令来直接连接Droplet。
      Docker Engine安装在了云服务器上，使用TLS认证通过TCP可远程连接到daemon.

2.  前往Digital Ocean控制台查看新建的Droplet.

    ![Droplet in Digital Ocean created with Machine](../images/ocean_droplet.png)

3.  在终端执行 `docker-machine ls`.

    ```
    $ docker-machine ls
    NAME             ACTIVE   DRIVER         STATE     URL                         SWARM
    default          *        virtualbox     Running   tcp://192.168.99.100:2376
    docker-sandbox   -        digitalocean   Running   tcp://45.55.139.48:2376
    ```

    注意新的云服务器已经运行但尚未激活。我们终端目前连接的仍然是默认机器，激活状态的机器ACTIVE栏会标记星号（*）

4.  执行 `docker-machine env docker-sandbox` 可以得到远程机器的环境变量，执行 `eval` 这行将当前的终端转为连接到 `docker-sandbox`.
    
    ```
    $ docker-machine env docker-sandbox
    export DOCKER_TLS_VERIFY="1"
    export DOCKER_HOST="tcp://45.55.222.72:2376"
    export DOCKER_CERT_PATH="/Users/victoriabialas/.docker/machine/machines/docker-sandbox"
    export DOCKER_MACHINE_NAME="docker-sandbox"
    # Run this command to configure your shell:
    # eval "$(docker-machine env docker-sandbox)"

    $ eval "$(docker-machine env docker-sandbox)"
    ```

5.  再次执行 `docker-machine ls` 验证下新的服务器是否激活, ACTIVE列中有星号（*）为激活状态.

    ```
    $ docker-machine ls
    NAME             ACTIVE   DRIVER         STATE     URL                         SWARM
    default          -        virtualbox     Running   tcp://192.168.99.100:2376
    docker-sandbox   *        digitalocean   Running   tcp://45.55.222.72:2376
    ```

6.  执行 `docker-machine` 相关命令查看远程主机配置。例如，使用 `docker-machine ip <machine>` 获取主机IP，`docker-machine inspect <machine>` 列出主机详情

    ```
    $ docker-machine ip docker-sandbox
    104.131.43.236

    $ docker-machine inspect docker-sandbox
    {
        "ConfigVersion": 3,
        "Driver": {
        "IPAddress": "104.131.43.236",
        "MachineName": "docker-sandbox",
        "SSHUser": "root",
        "SSHPort": 22,
        "SSHKeyPath": "/Users/samanthastevens/.docker/machine/machines/docker-sandbox/id_rsa",
        "StorePath": "/Users/samanthastevens/.docker/machine",
        "SwarmMaster": false,
        "SwarmHost": "tcp://0.0.0.0:3376",
        "SwarmDiscovery": "",
        ...
    ```

7.  使用 `docker` 命令验证Docker Engine是否正确安装
    可以使用基本的测试如 `docker run hello-world`，或者更有意义的尝试，如在你的远程机器上运行一个容器化的web服务器
    这个例子中, `-p` 参数用来将 `nginx` 容器80端口暴露出来，使主机`docker-sandbox` 可以通过 `8000` 端口访问到它.

    ```
    $ docker run -d -p 8000:80 --name webserver kitematic/hello-world-nginx
    Unable to find image 'kitematic/hello-world-nginx:latest' locally
    latest: Pulling from kitematic/hello-world-nginx
    a285d7f063ea: Pull complete
    2d7baf27389b: Pull complete
    ...
    Digest: sha256:ec0ca6dcb034916784c988b4f2432716e2e92b995ac606e080c7a54b52b87066
    Status: Downloaded newer image for kitematic/hello-world-nginx:latest
    942dfb4a0eaae75bf26c9785ade4ff47ceb2ec2a152be82b9d7960e8b5777e65
    ```

    在浏览器中输入 `http://<host_ip>:8000` 访问web服务器主页. 之前步骤使用 `docker-machine ip <machine>` 命令可以获取 `<host_ip>`. 端口使用 `docker run` 中暴露的端口.

    ![nginx webserver](../images/nginx-webserver.png)

#### 了解创建命令的默认和可选参数

为了方便操作, `docker-machine` 使用一些默认设置，如服务器基础镜像, 你可以分别设置相应参数来改变默认的设置 (如. `--digitalocean-image`). 例如, 如果你希望创建一台多核大内存的高配置云服务器(默认地 `docker-machine` 创建一台低配置机器)这些参数将很有帮助. 查看完整的参数/设置 可选值和默认值列表, 可以使用 `docker-machine create -h` 命令.
你也可以在Docker Machine 文档中查看 <a href="https://docs.docker.com/machine/drivers/os-base/" target="_blank">驱动参数和操作系统默认值</a> 及 <a href="https://docs.docker.com/machine/reference/create/" target="_blank">create</a> 命令


### 步骤 5. 使用Machine移除Droplet

移除主机及上面的所有容器和镜像，首先停止machine，然后执行 `docker-machine rm`:

    $ docker-machine stop docker-sandbox
    $ docker-machine rm docker-sandbox
    Do you really want to remove "docker-sandbox"? (y/n): y
    Successfully removed docker-sandbox

    $ docker-machine ls
    NAME      ACTIVE   DRIVER       STATE     URL                         SWARM
    default   *        virtualbox   Running   tcp:////xxx.xxx.xx.xxx:xxxx

当你执行上述命令时，在Digital Ocean控制台会看到相应的Droplet先停止然后被移除

如果你使用Docker Machine创建主机，但在云服务商管理控制台中移除主机，Machine会失去对服务器状态的跟踪。所以如果使用 `docker-machine --create`创建，请使用 `docker-machine rm` 删除。

## 继续阅读

* [Docker Machine驱动参考](/machine/drivers/)

* [Docker Machine概述](/machine/overview/)

* [使用 Docker Machine创建主机](/machine/get-started-cloud/)

* [安装 Docker Engine](../../installation/index.md)

* [Docker 用户指南](../../userguide/intro.md)