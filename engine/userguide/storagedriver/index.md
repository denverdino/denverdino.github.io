---
description: 如何为你的容器选择合适的存储驱动  
keywords: container, storage, driver, AUFS, btfs, devicemapper,zvfs
title: Docker 存储驱动
---

Docker 通过一些存储驱动的技术去管理镜像和容器的存储，这一章包含以下章节：


* [理解 镜像、容器和存储驱动](imagesandcontainers.md)
* [如何选择存储驱动](selectadriver.md)
* [AUFS存储驱动实践](aufs-driver.md)
* [Btrfs 存储驱动实践](btrfs-driver.md)
* [Device Mapper 存储驱动实践](device-mapper-driver.md)
* [OverlayFS 实践](overlayfs-driver.md)
* [ZFS 存储实践](zfs-driver.md)

如果你对Docker container刚接触不久，那么你应该先读一下[理解 镜像、容器和存储驱动](imagesandcontainers.md)，它诠释了镜像、容器和存储驱动的核心概念和技术，并通过这些概念帮助你更好的使用存储的驱动。

### 鸣谢

Docker的存储驱动大部分由伟大的贡献者Nigel Poulton在Docker的Jérôme Petazzoni的一点点的帮助下创造的，在Nigel Poulton的工作之余，他还制做了 [IT知识系列视频](http://www.pluralsight.com/author/nigel-poulton)，和别人一起制作了这个播客
[In Tech We Trust podcast](http://intechwetrustpodcast.com/)，并且在他的[Twitter](https://twitter.com/nigelpoulton)也有相应的内容。


&nbsp;
