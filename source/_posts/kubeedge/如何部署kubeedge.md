```
title: kubeedge环境搭建
date: 2022-03-30
categories:
- kubeedge
tags:
- kubeedge
language: zh-CN
toc: true
```

> 仅介绍使用keadm进行部署

## 先决条件

云端：需要搭建K8S集群环境，参考[K8S搭建EdgeX环境](https://caixindi.github.io/EdgeX/k8s-edgex%E9%83%A8%E7%BD%B2/#more)

边端：需要安装docker，参考[docker安装](https://docs.docker.com/get-started/)

## 使用 Keadm 进行部署

Keadm 用于安装 KubeEdge 的云和边缘组件。它不负责安装 K8s 和运行时。

请参考[kubernetes-compatibility](https://github.com/kubeedge/kubeedge#kubernetes-compatibility)以确认**Kubernetes 兼容性**并确定要安装的 Kubernetes 版本。

<!--more-->

## 局限性

- 目前支持`keadm`Ubuntu 和 CentOS 操作系统。RaspberryPi 支持正在进行中。
- 需要超级用户权限（或 root 权限）才能运行。

## 安装 keadm

运行以下命令一键安装`keadm`。

> 云端和边端都需要安装

```
# docker run --rm kubeedge/installation-package:v1.10.0 cat /usr/local/bin/keadm > /usr/local/bin/keadm && chmod +x /usr/local/bin/keadm
```

## cloudcore安装

```bash
keadm init --kubeedge-version=1.9.2 --advertise-address="THE-EXPOSED-IP" --kube-config=/root/.kube/config
```

## edgecore安装

首先在云端获取`token`

```bash
keadm gettoken
```

获取`token`之后，在边端执行

```bash
keadm join --cloudcore-ipport="THE-EXPOSED-IP":10000 --token=27a37ef16159f7d3be8fae95d588b79b3adaaf92727b72659eb89758c66ffda2.eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE1OTAyMTYwNzd9.JBj8LLYWXwbbvHKffJBpPd5CyxqapRQYDIXtFZErgYE
```

> 云端和边端的kubeedge安装过程都需要访问github，如果待部署环境不能访问github或者速度较慢，可以依靠科学手段从github上下载安装文件至/etc/kubeedge下，[下载链接](https://github.com/kubeedge/kubeedge/releases)。

### 故障解决

#### K8S安装tips

使用如下命令可以查看创建集群需要的镜像版本

```bash
kubeadm config images list
## 可以使用国内镜像仓库拉取 比如阿里云镜像仓库
kubeadm config images pull --image-repository registry.aliyuncs.com/google_containers --kubernetes-version v1.21.3
```

#### 解决error: system validation failed - Following Cgroup subsystem not mounted: [memory]

```bash
#上面加入kubeedge集群管理, 报错:
 Failed to start container manager, err: system validation failed - Following Cgroup subsystem not mounted: [memory]
E0125 16:42:09.131414    1655 edged.go:291] initialize module error: system validation failed - Following Cgroup subsystem not mounted: [memory]

#解决问题
#修改/boot/cmdline.txt
sudo vim /boot/cmdline.txt
cgroup_enable=memory cgroup_memory=1
#添加在同一行的最后面,接着内容后空格后添加, 注意:不要换行添加
#重启机器配置生效
reboot
```

#### 解决failed to run Kubelet: failed to create kubelet: misconfiguration: kubelet cgroup driver: "cgroupfs" is different from docker cgroup driver: "systemd"

失败的原因是你的dock 运行时的cgroup driver 和 kubelet 的 cgroup driver 运行方式是不一样的，需要将两者该为一致，此处修改docker：

```bash
/etc/docker/daemon.json
#只需要更改如下字段即可
"exec-opts": ["native.cgroupdriver=cgroupfs"]
```

#### 阿里云ECS error：Error while dialing dial tcp 127.0.0.1:2379: connect: connection refused". Reconnecting...

阿里云ECS安装需要使用私网IP
