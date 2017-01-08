---
description: Using DSC to configure a new Docker host
keywords: powershell, dsc, installation, usage, docker,  documentation
redirect_from:
- /engine/articles/dsc/
title: Use PowerShell DSC
---

Windows PowerShell Desired State Configuration (DSC)是一种配置管理工具，可扩展Windows PowerShell的现有功能。DSC使用声明性语法来定义配置应该处于的状态。有关PowerShell DSC的详细信息，请参见
[http://technet.microsoft.com/en-us/library/dn249912.aspx](http://technet.microsoft.com/en-us/library/dn249912.aspx).

## 前提条件


要使用本指南，您需要具有PowerShell v4.0或更高版本的Windows主机。

因为所包含的DSC配置脚本也使用官方PPA，所以目前也支持Ubuntu。Ubuntu主机上必须已经需要安装了OMI Server和PowerShell DSC for Linux。更多的信息请查看[https://github.com/MSFTOSSMgmt/WPSDSCLinux](https://github.com/MSFTOSSMgmt/WPSDSCLinux)。下面的github代码库也包括了PowerShell DSC for Linux的安装方法和初始化脚本。


## 安装


DSC配置示例源代码位于以下存储库中:
[https://github.com/anweiss/DockerClientDSC](https://github.com/anweiss/DockerClientDSC). 可以通过下面的方法下载代码:

    $ git clone https://github.com/anweiss/DockerClientDSC.git

## 使用方法

DSC配置使用一组shell脚本来确定是否在目标节点上配置指定的Docker组件。示例源代码库还包括一个脚本（`RunDockerClientConfig.ps1`），
这个脚本用于建立所需的CIM会话并执行

`Set-DscConfiguration` cmdlet。


更多信息请查看
[https://github.com/anweiss/DockerClientDSC](https://github.com/anweiss/DockerClientDSC).

### 安装Docker
Docker安装步骤:

```
apt-key adv --keyserver hkp://p80.pool.sks-keyservers.net:80 --recv-keys\
36A1D7869245C8950F966E92D8576A8BA88D21E9
sh -c "echo deb https://apt.dockerproject.org/repo ubuntu-trusty main\
> /etc/apt/sources.list.d/docker.list"
apt-get update
apt-get install docker-engine
```

确保您当前的工作目录设置为“DockerClientDSC”源并将DockerClient配置加载到当前的PowerShell中会话

```powershell
. .\DockerClient.ps1
```

为目标节点生成所需的DSC配置文件.mof

```powershell
DockerClient -Hostname "myhost"
```

利用DSC可以指定，修改配置，比如主机名


```powershell
DockerClient -ConfigurationData .\DockerConfigData.psd1
```

在指定机器上运行配置程序

```powershell
.\RunDockerClientConfig.ps1 -Hostname "myhost"
```

`RunDockerClientConfig.ps1`脚本还可以解析DSC配置数据文件并对多个节点执行配置:

```powershell
.\RunDockerClientConfig.ps1 -ConfigurationData .\DockerConfigData.psd1
```

### 容器镜像
镜像配置等同于: `docker pull [image]` 或者
`docker rmi -f [IMAGE]`.

使用和上面的相同步骤，执行`DockerClient`命令并结合`Image`参数:

```powershell
DockerClient -Hostname "myhost" -Image "node"
.\RunDockerClientConfig.ps1 -Hostname "myhost"
```

可以下载多个容器镜像:

```powershell
DockerClient -Hostname "myhost" -Image "node","mongo"
.\RunDockerClientConfig.ps1 -Hostname "myhost"
```

请用如下散列表，删除容器镜像:

```powershell
DockerClient -Hostname "myhost" -Image @{Name="node"; Remove=$true}
.\RunDockerClientConfig.ps1 -Hostname $hostname
```

### 容器  
容器配置等同于:

```
docker run -d --name="[containername]" -p '[port]' -e '[env]' --link '[link]'\
'[image]' '[command]'
```
or

```
docker rm -f [containername]
```
要创建或删除容器，您可以使用一个“Container”参数以及一个或多个哈希表。传递给此参数的散列表可以具有以下属性：

- Name (required)
- Image (required unless Remove property is set to `$true`)
- Port
- Env
- Link
- Command
- Remove

例如， 为你的容器创建以下配置：

```powershell
$webContainer = @{Name="web"; Image="anweiss/docker-platynem"; Port="80:80"}
```

再用与上面相同的步骤，执行`DockerClient`命令，这个命令需要跟着`-Image` and `-Container`

```powershell
DockerClient -Hostname "myhost" -Image node -Container $webContainer
.\RunDockerClientConfig.ps1 -Hostname "myhost"
```

删除已有容器:

```powershell
$containerToRemove = @{Name="web"; Remove=$true}
DockerClient -Hostname "myhost" -Container $containerToRemove
.\RunDockerClientConfig.ps1 -Hostname "myhost"
```

这里有一个用来创建容器的例子：

```powershell
$containerProps = @{Name="web"; Image="node:latest"; Port="80:80"; `
Env="PORT=80"; Link="db:db"; Command="grunt"}
```
