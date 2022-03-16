---
title: 树莓派系统部署k3s有关操作
date: 2021-10-27
categories:
- EdgeX
tags:
- EdgeX部署
language: zh-CN
toc: true
---

## 开启Wifi模块

```shell
#开启wifi模块
rfkill unblock wifi
sudo iwlist scan | grep ESSID
```

<!--more-->

### 配置host

```sh
192.168.137.100          master
192.168.137.101          agent01
192.168.137.102          agent02
```

### 配置wifi以及静态ip

```shell
sudo nano /etc/wpa_supplicant/wpa_supplicant.conf

ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1
#在结尾处添加
network={
        ssid="TestWifi"
        psk="12345678"
}
sudo nano /etc/dhcpcd.conf
#结尾处添加
interface wlan0
static ip_address=192.168.137.100/24  #需要重新配置
static routers=192.168.137.1
static domain_name_servers=114.114.114.114
```

## 在 Raspbian Buster 上启用旧版的 iptables[#](https://docs.rancher.cn/docs/k3s/advanced/_index#在-raspbian-buster-上启用旧版的-iptables)

Raspbian Buster 默认使用`nftables`而不是`iptables`。 **K3S** 网络功能需要使用`iptables`，而不能使用`nftables`。 按照以下步骤切换配置**Buster**使用`legacy iptables`：

```
sudo iptables -F
sudo update-alternatives --set iptables /usr/sbin/iptables-legacy
sudo update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy
```

## 为 Raspbian Buster 启用 cgroup[#](https://docs.rancher.cn/docs/k3s/advanced/_index#为-raspbian-buster-启用-cgroup)

标准的 Raspbian Buster 安装没有启用 `cgroups`。**K3S** 需要`cgroups`来启动 systemd 服务。在`/boot/cmdline.txt`中添加`cgroup_memory=1 cgroup_enable=memory`就可以启用`cgroups`。

```sh
reboot
```

### server节点

设置`K3S_URL`参数会使 K3s 以 worker 模式运行。K3s agent 将在所提供的 URL 上向监听的 K3s 服务器注册。`K3S_TOKEN`使用的值存储在你的服务器节点上的`/var/lib/rancher/k3s/server/node-token`路径下。

```sh
curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn K3S_NODE_NAME=master sh -
```

### 获取token

`K3S_TOKEN`使用的值存储在服务器节点上的`/var/lib/rancher/k3s/server/node-token`路径下

### agent节点

```sh
curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn K3S_URL=https://master:6443 K3S_TOKEN=K10742df0412ea48b58f29a8a0ece0e2aea625e306853ba92c47ba86870a57a25df::server:9dbe384cf66d9da0504cd79f6a644db6 K3S_NODE_NAME=agent02 sh -
```

### 禁止调度server

```
kubectl cordon server
```

由于是arm64架构，需要更改yaml文件镜像拉取源

### 给节点打标签

```sh
kubectl label node agent01 disktype=node1
```

