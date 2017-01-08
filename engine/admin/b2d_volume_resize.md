---
description: Resizing a Boot2Docker volume in VirtualBox with GParted
keywords: boot2docker, volume,  virtualbox
published: false
title: Resize a Boot2Docker volume
---

# 如何处理Boot2Docker抛出的错误"no space left on device"？

当你使用了大量的容器镜像，或者你使用的容器镜像占用了巨大的存储空间，这时下拉新的容器镜像会失败，并且抛出"no space left
on device"的错误。这是由于Boot2Docker数据卷已经填满了。此时，我们有两个解决方案可以尝试。


## 方案 1: 添加`DiskImage`到boot2docker的配置文件

`boot2docker`命令从`$BOOT2DOCKER_PROFILE`，`$BOOT2DOCKER_DIR/profile`或者`$HOME/.boot2docker/profile`加载配置（在Windows，是`%USERPROFILE%/.boot2docker/profile`）

1. 使用`boot2docker config`命令查看当前配置

        $ boot2docker config
        # boot2docker 的配置文件: /Users/mary/.boot2docker/profile
        Init = false
        Verbose = false
        Driver = "virtualbox"
        Clobber = true
        ForceUpgradeDownload = false
        SSH = "ssh"
        SSHGen = "ssh-keygen"
        SSHKey = "/Users/mary/.ssh/id_boot2docker"
        VM = "boot2docker-vm"
        Dir = "/Users/mary/.boot2docker"
        ISOURL = "https://api.github.com/repos/boot2docker/boot2docker/releases"
        ISO = "/Users/mary/.boot2docker/boot2docker.iso"
        DiskSize = 20000
        Memory = 2048
        CPUs = 8
        SSHPort = 2022
        DockerPort = 0
        HostIP = "192.168.59.3"
        DHCPIP = "192.168.59.99"
        NetMask = [255, 255, 255, 0]
        LowerIP = "192.168.59.103"
        UpperIP = "192.168.59.254"
        DHCPEnabled = true
        Serial = false
        SerialFile = "/Users/mary/.boot2docker/boot2docker-vm.sock"
        Waittime = 300
        Retries = 75

  这些配置是来自于`boot2docker`的配置文件，而这些正是`boot2docker`当前使用的配置。

2. 可以利用命令`boot2docker config > ~/.boot2docker/profile`初始化配置

3. 见下面的配置添加到`$HOME/.boot2docker/profile`:

        # 容器镜像的大小 MB
        DiskSize = 50000

4. 按以下顺序重启Boot2Docker，这样新的配置就启用了。

        $ boot2docker poweroff
        $ boot2docker destroy
        $ boot2docker init
        $ boot2docker up

## 解决方案 2：增大boot2docker的数据卷大小

This solution increases the volume size by first cloning it, then resizing it
using a disk partitioning tool. We recommend
我们推荐的[GParted](https://sourceforge.net/projects/gparted/files/)作为一个可引导的
ISO, 是一个免费下载，并与VirtualBox可以很好地一起工作。

1. 停止 Boot2Docker

  利用命令行停止Boot2Docker虚拟机:

      $ boot2docker stop

2. 克隆VMDK镜像成VDI映像

  Boot2Docker附带一个VMDK镜像，但它不能用VirtualBox的本地工具来调整。 我们会通过创造一个VDI镜像并克隆成VMDK镜像的方式创建它。

3. 通过virtual Box的命令行工具，将VMDK镜像克隆成VDI镜像:

        $ vboxmanage clonehd /full/path/to/boot2docker-hd.vmdk /full/path/to/<newVDIimage>.vdi --format VDI --variant Standard

4. 调整VDI的卷大小
  
  选择一个符合您要求的数据卷大小。如果您会使用大量的容器，或者您的容器占据大量存储，对您来说卷的大小越大越好：

      $ vboxmanage modifyhd /full/path/to/<newVDIimage>.vdi --resize <size in MB>

5. 下载磁盘分区工具ISO

  要调整卷大小，我们将使用 [GParted](https://sourceforge.net/projects/gparted/files/).请首先创建Boot2Docker VM IDE总线，并且下载ISO，将ISO添加到总线里。
  Once you've downloaded the tool, add the ISO to the Boot2Docker VM IDE bus.
  You might need to create the bus before you can add the ISO.

  > **注意:**
  > 请确保您选择的分区工具可以使Boot2Docker VMI能够被它引导启动

  <table>
      <tr>
          <td><img src="/articles/b2d_volume_images/add_new_controller.png"><br><br></td>
      </tr>
      <tr>
          <td><img src="/articles/b2d_volume_images/add_cd.png"></td>
      </tr>
  </table>

6. 添加新的VDI镜像

  从VirtualBox里Boot2Docker的镜像配置中的SATA controller里移除VMDK镜像，并添加VDI镜像。

  <img src="/articles/b2d_volume_images/add_volume.png">

7. 确认启动顺序
    
    在为Boot2Docker虚拟机的**系统**设置，确保CD / DVD是在启动顺序列表的顶部。

    <img src="/articles/b2d_volume_images/boot_order.png">

8. 启动磁盘分区工具 ISO

  在VirtualBox里，手动启动Boot2Docker虚拟机, 并且磁盘分区ISO应启动。 使用的gparted，选择的**GParted**运行（默认设置）选项。 选择默认的键盘，语言和XWindows的设置和GParted工具将启动并显示您所创建的VDI量。 右键单击VDI和选择**调整大小/移动**。

  <img src="/articles/b2d_volume_images/gparted.png">

9. 拖动滑动条到最大可用值。

10. 点击**Resize/Move** ，后点击**Apply**.

  <img src="/articles/b2d_volume_images/gparted2.png">

11. 退出GParted并关闭虚拟机.

12. 从VirtualBox的Boot2Docker虚拟机的IDE控制器中移除GParted ISO。

13. 启动 Boot2Docker 虚拟机
  
  在VirtualBox中手动启动Boot2Docker虚拟机。虚拟机应该可以自动登录，但如果没有，可以使用用户名/密码`docker/tcuser`。 使用df -h命令，验证更改是否生效。
  
  <img src="/articles/b2d_volume_images/verify.png">

顺利完成!
