基于Ubuntu18.04 

需要两个CPU核心

##### 第1步：主机名解析

好处就是节点之间可以通过主机名直接访问

```shell
$ vim /etc/hosts
```

添加内容，这里面的ip和主机名需要以自己的为准，比如：
192.168.0.200 master
192.168.0.201 node1
192.168.0.202 node2

##### 第2步：时间同步

```shell
$ sudo apt install chrony
$ systemctl start chronyd
$ systemctl enable chronyd
```

查看是否工作

```shell
$ date    #返回时间
$ systemctl is-enabled chronyd.service  #应该返回enable
```

##### 第3步：关闭交换区

```shell
$ swapoff -a  #这时临时的
#永久关闭,把里面swapfile那一行注释掉
$ vim /etc/fstab
```

##### 第4步:禁用Iptables和firewalld(我的系统并没有这预装这两个服务)

```shell
$ systemctl stop firewalld
#输出：Failed to stop firewalld.service: Unit firewalld.service not loaded.  说明防火墙没启动  不用管
$ systemctl disable firewalld
#输出：Failed to disable unit: Unit file firewalld.service does not exist. 说明防火墙根本不存在  不用管

#关闭系统iptables服务
$ systemctl stop iptables
$ systemctl disable iptables
#输出:Failed to stop iptables.service: Unit iptables.service not loaded.  不用管
#输出:Failed to disable unit: Unit file iptables.service does not exist.  不用管
```

##### 第5步：禁用selinux

```shell
#首先查看这个服务是否开启
$ getenforce
#如果输出：Command 'getenforce' not found, but can be installed with: apt install selinux-utils 如果是这样，那下面步骤不用做
$ vim /etc/selinux/config   #将文件里面的SELINUX=disabled  需要重启才生效，可以到最后进行重启
```

##### 第6步：编辑linux内核参数

```shell
$ vim /etc/sysctl.d/kubernetes.conf
#添加下面这些配置
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1

#重新加载配置
$ sysctl -p
#加载网桥过滤模块
$ modprobe br_netfilter
#查看是否加载成功
$ lsmod | grep br_netfilter
#系统重启
$ reboot
```

##### 第6步：配置ipvs

```shell
#安装ipset和ipvsadm
$ apt install ipset
$ apt install ipvsadm
#加载需要的模块
$ mkdir -p  /etc/sysconfig/modules/
cat <<EOF> /etc/sysconfig/modules/ipvs.modules 
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
EOF
#添加可执行权限
$ chmod +x /etc/sysconfig/modules/ipvs.modules
$ bash /etc/sysconfig/modules/ipvs.modules
#查看是否加载成功
$ lsmod | grep -e ip_vs
```

##### 第7步：安装docker

手动安装步骤省略，可以直接使用脚本安装

```shell
curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun
```

安装完之后执行以下命令

```shell
#创建 /etc/docker 目录
$ mkdir /etc/docker
#配置镜像源以及cgroup
cat > /etc/docker/daemon.json <<EOF
{
"exec-opts": ["native.cgroupdriver=systemd"],
"registry-mirrors": ["https://fsmye2b3.mirror.aliyuncs.com"]
}
EOF
```

重启docker服务

```shell
$ systemctl daemon-reload 
$ systemctl restart docker 
$ systemctl enable docker
```

##### 第8步：安装kubernetes组件

```shell
#添加apt-key和源
$ sudo apt update && sudo apt install -y apt-transport-https curl
$ curl -s https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add -
$ echo "deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main" >>/etc/apt/sources.list.d/kubernetes.list
#安装
$ sudo apt update
$ sudo apt-get install -y kubelet=1.22.1-00 kubeadm=1.22.1-00 kubectl=1.22.1-00
$ sudo apt-mark hold kubelet=1.22.1-00 kubeadm=1.22.1-00 kubectl=1.22.1-00
#配置kubelet的cgroup
$ vim /etc/sysconfig/kubelet
#添加如下内容
KUBELET_CGROUP_ARGS="--cgroup-driver=systemd"
KUBE_PROXY_MODE="ipvs"
#设置开机自启
$ systemctl enable kubelet
```

##### 第9步：准备集群环境相关镜像

编写脚本，设置镜像仓库

```shell
$ vim pull_k8s_images.sh
```

下面的内容粘贴进去

```
set -o errexit
set -o nounset
set -o pipefail
##定义版本
KUBE_VERSION=v1.22.1
KUBE_PAUSE_VERSION=3.5
ETCD_VERSION=3.5.0-0
DNS_VERSION=v1.8.4

GCR_URL=k8s.gcr.io
##使用的镜像仓库
DOCKERHUB_URL=k8simage
##镜像列表
images=(
kube-proxy:${KUBE_VERSION}
kube-scheduler:${KUBE_VERSION}
kube-controller-manager:${KUBE_VERSION}
kube-apiserver:${KUBE_VERSION}
pause:${KUBE_PAUSE_VERSION}
etcd:${ETCD_VERSION}
coredns:${DNS_VERSION}
)
##拉取并改名
for imageName in ${images[@]} ; do
  docker pull $DOCKERHUB_URL/$imageName
  docker tag $DOCKERHUB_URL/$imageName $GCR_URL/$imageName
  docker rmi $DOCKERHUB_URL/$imageName
done
```

```shell
#授予执行权限
$ chmod +x ./pull_k8s_images.sh
#执行脚本
$ ./pull_k8s_images.sh
```

这个时候使用docker images查看拉取镜像是否成功，应该唯独没有coredns 因为这个仓库里面没有
接下来单独拉取coredns

```shell
$ docker pull coredns/coredns:1.8.4
$ docker tag coredns/coredns:1.8.4 k8s.gcr.io/coredns/coredns:v1.8.4
$ docker rmi coredns/coredns:1.8.4
```

这个时候使用docker images检查一遍，应该都有了

**如果部署多个结点，上面的步骤需要在每个节点上都运行一遍**



##### 第10步:集群初始化（在master节点操作）

```shell
#这边的--apiserver-advertise-address=192.168.0.150需要根据自己master节点的ip进行修改
$ sudo kubeadm init --apiserver-advertise-address=192.168.0.150 --pod-network-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/12 --kubernetes-version=v1.22.1
#创建配置文件夹
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
#子节点加入集群，在子节点操作，下面的命令会在master 执行init操作之后终端会有输出（每个人不一样，需要用自己的），直接复制即可，比如：
kubeadm join 192.168.0.150:6443 --token b0oxg7.zxc5etvu2412etn1 \
	--discovery-token-ca-cert-hash sha256:a507d34b2f40d931edf35f0dac6b95ca3f68cf529d30557b3f7258cd955fa75e 
```

##### 第11步：部署网络（在master节点操作）

由于raw.githubusercontent.com被墙，先配置host
这边需要在https://www.ipaddress.com/查询raw.githubusercontent.com的ip地址

```shell
$ vim /etc/hosts
#添加如下内容
185.199.108.133 raw.githubusercontent.com
#执行（如果网络还是不行，可以下载这个文件传到虚拟机上）
$ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

使用kubectl get nodes命令查看，节点应该都是ready状态

```shell
$ kubectl get nodes
```

部署edgex，可以用1.x版本也可以用2.0

```shell
$ vim k8s-redis-no-secty-with-ui.yaml
```

粘贴内容，文件群里有，如果用1.x版本的，官网有，地址如下：https://github.com/edgexfoundry/edgex-examples/blob/main/deployment/kubernetes/k8s-hanoi-redis-no-secty.yml

```shell
#允许master节点部署pod,这边如果只创建了一个master节点，没有工作节点，则需要设置；如果有工作节点，则跳过此步骤
$ kubectl taint nodes --all node-role.kubernetes.io/master-
#部署（第一次耗时会比较长）
$ kubectl apply -f k8s-redis-no-secty-with-ui.yaml 
```

最后可以通过kubectl get pods,svc 查看状态
结束

