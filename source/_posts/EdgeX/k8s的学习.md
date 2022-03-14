---
title: k8s的学习
categories:
- EdgeX
tags:
- EdgeX构建
language: zh-CN
toc: true
---

> k8s的管理功能体现在什么地方？

​	早期应用程序的部署会直接部署在物理机上，较为简单，但不同的应用程序之间会产生影响。之后产生了虚拟化部署，虽然保证了程序间的隔离，但极大地耗费资源。于是容器化部署出现了，容器化部署可以保证每个容器拥有自己的文件系统、CPU、内存、进程空间等，运行应用程序所需要的资源都被容器包装，并与底层基础架构解耦等。

<!--more-->

​	具体来说，容器化部署有如下这些优点（来自k8s官方文档）：

- 敏捷应用程序的创建和部署：与使用 VM 镜像相比，提高了容器镜像创建的简便性和效率。
- 持续开发、集成和部署：通过快速简单的回滚（由于镜像不可变性），支持可靠且频繁的 容器镜像构建和部署。
- 关注开发与运维的分离：在构建/发布时而不是在部署时创建应用程序容器镜像， 从而将应用程序与基础架构分离。
- 可观察性：不仅可以显示操作系统级别的信息和指标，还可以显示应用程序的运行状况和其他指标信号。
- 跨开发、测试和生产的环境一致性：在便携式计算机上与在云中相同地运行。
- 跨云和操作系统发行版本的可移植性：可在 Ubuntu、RHEL、CoreOS、本地、 Google Kubernetes Engine 和其他任何地方运行。
- 以应用程序为中心的管理：提高抽象级别，从在虚拟硬件上运行 OS 到使用逻辑资源在 OS 上运行应用程序。
- 松散耦合、分布式、弹性、解放的微服务：应用程序被分解成较小的独立部分， 并且可以动态部署和管理 - 而不是在一台大型单机上整体运行。
- 资源隔离：可预测的应用程序性能。
- 资源利用：高效率和高密度。

![部署演进](https://cxd-note-img.oss-cn-hangzhou.aliyuncs.com/typora-note-img/container_evolution.svg)

但是容器化部署的方式也会遇到一些问题，比如：

- 一个容器故障停机了，怎么样让另外一个容器立刻启动去替补停机的容器
- 当并发访问量变大的时候，怎么样做到横向扩展容器数量

这样的问题便是容器编排问题，由此也产生了一系列的容器编排管理工具，比如Swarm，Kubernetes。

总的来说，k8s的管理功能包括如下几个方面：

- **自我修复**：一旦某一个容器崩溃，能够迅速启动新的容器（秒级）
- **弹性伸缩**：可以根据需要，自动对集群中正在运行的容器数量进行调整
- **服务发现**：服务可以通过自动发现的形式找到它所依赖的服务
- **负载均衡**：如果一个服务起动了多个容器，能够自动实现请求的负载均衡并分配网络流量
- **版本回退**：如果发现新发布的程序版本有问题，可以立即回退到原来的版本
- **存储编排**：可以根据容器自身的需求自动创建挂载存储卷。（持久化容器内数据）

![Kubernetes 组件](https://cxd-note-img.oss-cn-hangzhou.aliyuncs.com/typora-note-img/components-of-kubernetes.svg)

> Edgex2.0爱尔兰版本部署

yaml文件需要注意的问题：全局环境变量和局部环境变量需要全部大写（目前是这样的）

kuiper挂载的目录问题：目前还不知道具体的挂载目录是哪个（暂时不实现数据持久化），用/kuiper/data会出现问题 找不到对应的kv表

![](https://cxd-note-img.oss-cn-hangzhou.aliyuncs.com/typora-note-img/image-20210918191542159-163428787658730.png)

![](https://cxd-note-img.oss-cn-hangzhou.aliyuncs.com/typora-note-img/image-20210918191542159-163428787658730.png)

一主一从：

![](https://cxd-note-img.oss-cn-hangzhou.aliyuncs.com/typora-note-img/image-20210918184416860.png)

部署：

![](https://cxd-note-img.oss-cn-hangzhou.aliyuncs.com/typora-note-img/image-20210918184506559-163428789042732.png)

查看pods是否工作正常：

![](https://cxd-note-img.oss-cn-hangzhou.aliyuncs.com/typora-note-img/image-20210918184506559-163428789042732.png)

通过ui页面：

![](https://cxd-note-img.oss-cn-hangzhou.aliyuncs.com/typora-note-img/image-20210918184506559-163428789042732.png)

虚拟设备自动事件工作正常：

![](https://cxd-note-img.oss-cn-hangzhou.aliyuncs.com/typora-note-img/image-20210918184826689.png)

