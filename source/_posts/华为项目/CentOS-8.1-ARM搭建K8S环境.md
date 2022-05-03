---
title: 华为项目
date: 2022-05-03
categories:
- 华为项目
tags:
- 虚拟化
language: zh-CN
toc: true
---

### Containerd 1.4在CentOS 8.1下的兼容性验证

#### 1. 安装go

#### 2. 克隆release1.4分支

```shell
git clone  https://github.com/containerd/containerd -b release/1.4 
```

#### 3. 安装protoc

```shell
wget -c https://github.com/protocolbuffers/protobuf/releases/download/v3.11.4/protoc-3.11.4-linux-x86_64.zip
sudo unzip protoc-3.11.4-linux-x86_64.zip -d /usr/local
```

#### 4. 安装Btrfs

```shell
yum install btrfs-progs-devel
```

#### 5. 安装runc

```shell
go install github.com/opencontainers/runc@1.1.1
```

#### 6. 安装containerd

```shell
cd containerd
go mod init github.com/containerd/containerd
go mod tidy
go mod vendor
make
smake install
```

#### 7. 配置systemd

```shell
cp containerd.service /usr/local/lib/systemd/system/containerd.service
systemctl daemon-reload
systemctl enable --now containerd
```

#### 8. 生成containerd的默认配置文件并配置cgroup

```shell
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml

# 使用 systemd cgroup 驱动程序 
# 在 /etc/containerd/config.toml 中设置

[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  ...
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true

# 重新启动 containerd   
sudo systemctl restart containerd
```

#### 9. 安装K8S

##### 1. 关闭防火墙

```shell
systemctl stop firewalld && systemctl disable firewalld
```

##### 2. 关闭selinux

```shell
setenforce 0
sed -i "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config
```

##### 3. 关闭交换分区

```shell
# 临时关闭
swapoff -a
#注释掉/etc/fstab中的swap
vi /etc/fstab
#/dev/mapper/centos-swap     swap      swap     defaults     0     0
#确认swap是否被禁用（输出空值即为禁用）
cat /proc/swaps
Filename       
```

##### 4. 加载内核模块并且设置sysctl 参数

```shell
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# 设置必需的 sysctl 参数，这些参数在重新启动后仍然存在。
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

# 应用 sysctl 参数而无需重新启动
sudo sysctl --system
```

##### 5. 允许iptables检查桥接流量

```shell
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```

##### 6. 安装kubernetes组件

```shell
# 配置kubernetes软件源
cat > /etc/yum.repos.d/kubernetes.repo <<EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-aarch64/ 
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
# 安装kubelet kubeadm kubectl组件
yum install -y kubelet-1.20.6 kubeadm-1.20.6 kubectl-1.20.6
```

##### 7. 配置kubelet使用containerd作为容器运行时，指定cgroupDriver为systemd模式

```
cat > /etc/sysconfig/kubelet <<EOF
KUBELET_EXTRA_ARGS=--cgroup-driver=systemd
EOF
```

##### 8. 设置kubelet开机自启动

```shell
systemctl enable --now kubelet && systemctl restart kubelet
```

##### 9. 修改config.toml文件中的sandbox_image、endpoint、systemd cgroup参数

```shell
vim /etc/containerd/config.toml
 
[plugins."io.containerd.grpc.v1.cri"] 字段下的sandbox_image修改为如下
sandbox_image="registry.aliyuncs.com/google_containers/pause:3.2"
 
[plugins."io.containerd.grpc.v1.cri".registry]字段下的endpoint修改为如下
endpoint = ["https://registry.cn-hangzhou.aliyuncs.com"]
```

##### 10. 安装CRI客户端工具crictl

```shell
wget https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.20.0/crictl-v1.20.0-linux-arm64.tar.gz
 
sudo tar zxvf crictl-v1.20.0-linux-amd64.tar.gz -C /usr/local/bin
 
cat > /etc/crictl.yaml <<EOF
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
EOF

#或者执行以下语句添加参数
crictl config runtime-endpoint unix:/run/containerd/containerd.sock
```

##### 11. 初始化集群

```shell
sudo kubeadm init --image-repository registry.aliyuncs.com/google_containers --apiserver-advertise-address=10.208.55.173 --pod-network-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/12 --kubernetes-version=v1.20.6

# 初始化之后执行以下操作
#非root用户
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
 
#root用户
export KUBECONFIG=/etc/kubernetes/admin.conf
```

##### 12. 部署flannel组件

```shell
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```







### Prometheus 2.32.0在CentOS 8.1下的兼容性验证

















### lstio 组件下载安装指导

#### 1.下载并解压Istio

```shell
$ wget https://github.com/istio/istio/releases/download/1.8.1/istio-1.8.1-linux-arm64.tar.gz
$ tar -xzvf istio-1.8.0-linux-arm64.tar.gz
```

#### 2.移动到Istio包目录

```shell
$ cd istio-1.8.1
```

#### 3.将istioctl添加至环境变量

```shell
$ export PATH=$PWD/bin:$PATH
```

#### 4.安装istio Operator

```shell
$ istioctl operator init --hub=docker.io/querycapistio 
```

5.安装istio

```shell
$ kubectl create ns istio-system
$ kubectl apply -f demo.yaml
```

`demo.yaml`

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: arm64-affinity-patch
  namespace: istio-system
spec:
  hub: docker.io/querycapistio
  components:
    pilot:
      enabled: false
    ingressGateways:
    - namespace: istio-system
      name: istio-ingressgateway
      enabled: true
      k8s:
        affinity:
          nodeAffinity:
            preferredDuringSchedulingIgnoredDuringExecution:
            - preference:
                matchExpressions:
                - key: kubernetes.io/arch
                  operator: In
                  values:
                  - arm64
              weight: 2
            requiredDuringSchedulingIgnoredDuringExecution:
              nodeSelectorTerms:
              - matchExpressions:
                - key: kubernetes.io/arch
                  operator: In
                  values:
                  - arm64
    egressGateways:
    - namespace: istio-system
      name: istio-egressgateway
      enabled: true
      k8s:
        affinity:
          nodeAffinity:
            preferredDuringSchedulingIgnoredDuringExecution:
            - preference:
                matchExpressions:
                - key: kubernetes.io/arch
                  operator: In
                  values:
                  - arm64
              weight: 2
            requiredDuringSchedulingIgnoredDuringExecution:
              nodeSelectorTerms:
              - matchExpressions:
                - key: kubernetes.io/arch
                  operator: In
                  values:
                  - arm64
```

### listio 组件基本功能验证报告