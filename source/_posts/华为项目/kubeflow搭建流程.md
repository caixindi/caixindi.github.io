---
title: Kubeflow集群环境搭建流程
date: 2022-08-30
categories:
- 华为项目
tags:
- kubeflow
language: zh-CN
toc: true
---

## Kubeflow集群环境搭建流程

### 先决条件

- 科学上网

- kubernetes(<=1.21)
- kustomize(3.2.0) [下载链接](https://github.com/kubernetes-sigs/kustomize/releases/tag/v3.2.0)
- kubectl

> 允许master节点被调度
>
> ```shell
> kubectl taint nodes --all node-role.kubernetes.io/master-
> ```

<!--more-->

### 安装

#### 步骤1 克隆官方 manifests 仓库

```shell
git clone https://github.com/kubeflow/manifests.git -b v1.5-branch
```

#### 步骤2 创建持久卷

```shell
sudo rm -rf /mnt/pv{1..4}
sudo mkdir /mnt/pv{1..4}
kubectl create -f - <<EOF
kind: PersistentVolume
apiVersion: v1
metadata:
  name: pv-volume1
spec:
  storageClassName:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/pv1"
---
kind: PersistentVolume
apiVersion: v1
metadata:
  name: pv-volume2
spec:
  storageClassName:
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/pv2"
---
kind: PersistentVolume
apiVersion: v1
metadata:
  name: pv-volume3
spec:
  storageClassName:
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/pv3"
---
kind: PersistentVolume
apiVersion: v1
metadata:
  name: pv-volume4
spec:
  storageClassName:
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/pv4"
EOF
```

#### 步骤3 修改 authservice 有关配置

配置文件路径`common/oidc-authservice/base/statefulset.yaml`

```shell
cat > common/oidc-authservice/base/statefulset.yaml << EOF
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: authservice
spec:
  replicas: 1
  selector:
    matchLabels:
      app: authservice
  serviceName: authservice
  template:
    metadata:
      annotations:
        sidecar.istio.io/inject: "false"
      labels:
        app: authservice
    spec:
      initContainers:
      - name: fix-permission
        image: busybox
        command: ['sh', '-c']
        args: ['chmod -R 777 /var/lib/authservice;']
        volumeMounts:
        - mountPath: /var/lib/authservice
          name: data
      containers:
      - name: authservice
        image: gcr.io/arrikto/kubeflow/oidc-authservice:6ac9400
        imagePullPolicy: IfNotPresent
        ports:
        - name: http-api
          containerPort: 8080
        envFrom:
          - secretRef:
              name: oidc-authservice-client
          - configMapRef:
              name: oidc-authservice-parameters
        volumeMounts:
          - name: data
            mountPath: /var/lib/authservice
        readinessProbe:
            httpGet:
              path: /
              port: 8081
      securityContext:
        fsGroup: 111
      volumes:
        - name: data
          persistentVolumeClaim:
              claimName: authservice-pvc
EOF
```

#### 步骤4 使用单个命令安装

```shell
while ! kustomize build example | kubectl apply -f -; do echo "Retrying to apply resources"; sleep 10; done
```

#### 步骤5 登录集群

安装后，所有 Pod 都需要一些时间才能准备就绪。在尝试连接之前，请确保所有 Pod 都已准备就绪，否则可能会遇到意外错误。要检查所有与 Kubeflow 相关的 Pod 是否已准备就绪，请使用以下命令：

```shell
kubectl get pods -n cert-manager
kubectl get pods -n istio-system
kubectl get pods -n auth
kubectl get pods -n knative-eventing
kubectl get pods -n knative-serving
kubectl get pods -n kubeflow
kubectl get pods -n kubeflow-user-example-com
```

#### 端口转发

访问 Kubeflow 的默认方式是通过端口转发。这能够快速开始，而不会对环境有任何要求。运行以下命令将 Istio 的 Ingress-Gateway 端口转发到本地端口`8080`：

```
kubectl port-forward svc/istio-ingressgateway -n istio-system 8080:80
```

> 后台运行 
>
> ```shell
> nohup kubectl port-forward svc/istio-ingressgateway -n istio-system 8080:80 > kubeflow-ui-log.txt 2>&1 &
> ```

运行命令后，通过执行以下操作访问 Kubeflow Central Dashboard：

1. 打开浏览器并访问`http://localhost:8080`，会看到 Dex 登录屏幕。
2. 使用默认用户的凭据登录。默认电子邮件地址是`user@example.com`，默认密码是`12341234`。