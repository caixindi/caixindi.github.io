---
title: K3s的安装
categories:
- EdgeX
tags:
- EdgeX构建
language: zh-CN
toc: true
---

### k3s安装

k3s的安装步骤极为简单，只需一步就可以完成安装：

```shell
curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn sh -
```

但其实在k3s-install.sh文件中做了许多的配置工作。

将工作节点添加到集群，执行下面的步骤：

<!--more-->

```shell
curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn K3S_URL=https://myserver:6443 K3S_TOKEN=mynodetoken sh -
```

> k3s比k8s好在哪些地方

- 其二进制文件只有60MB左右，并且需要的内存比k8s小得多，可以在 512MB 内存以上的任何机器上运行集群，这意味着我们可以允许 Pod 运行在主节点和节点上。
- 适合边缘计算（单主节点）
- 简单的安装方式/并且可以快速启动集群
- 缩减了外部依赖项，并且所有的依赖项都在应该单一的二进制文件中