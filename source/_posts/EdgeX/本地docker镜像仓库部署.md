## 1.运行本地注册表

使用如下命令启动注册表容器：

```
$ docker run -d -p 5000:5000 --restart=always --name registry registry:2
```

> **警告**:仅适用于测试的注册表配置。生产就绪的注册表必须受 TLS 保护，并且最好使用访问控制机制。继续阅读并继续阅读[配置指南](https://docs.docker.com/registry/configuration/)以部署生产就绪注册表。

## 2.从dockerhub上拉取镜像后标记并且推送到本地注册表

从 Docker Hub 拉取镜像并将其推送到注册表。以下示例`nginx:latest`从 Docker Hub拉取映像并将其重新标记为`nginx:test`，然后将其推送到本地注册表。最后， 移除`nginx:latest`和`nginx:test`并且从本地拉取`nginx:test`。

1. `nginx:latest`从 Docker Hub拉取镜像。

   ```
   $ docker pull nginx:latest
   ```

2. 将图像标记为`localhost:5000/my-ubuntu`。这会为现有图像创建一个附加标签。当标签的第一部分是主机名和端口时，Docker 在推送时将其解释为注册表的位置。

   ```
   $ docker tag nginx:latest localhost:5000/nginx:test
   ```

3. 将映像推送到在以下位置运行的本地注册表`localhost:5000`：

   ```
   $ docker push localhost:5000/nginx:test
   ```

4. 删除本地缓存`ubuntu:16.04`和`localhost:5000/my-ubuntu` 图像，以便您可以测试从注册表中拉取图像。这不会`localhost:5000/my-ubuntu`从您的注册表中删除图像。

   ```
   $ docker image remove nginx:latest
   $ docker image remove localhost:5000/nginx:test
   ```

5. `localhost:5000/my-ubuntu`从本地注册表中拉取映像。

   ```
   $ docker pull localhost:5000/nginx:test
   ```

> 这个步骤可能会出现 docker registry push 遇到 “Get https://xxx:5000/v2/: http: server gave HTTP response to HTTPS client ”的问题

### 解决方案

#### a.修改docker配置文件

```sh
vi /etc/docker/daemon.json
#添加 insecure-registries
{
    "insecure-registries": ["http://xxx.xxx.xxx.xxx:5000"]
}
```

#### b.重新加载并且重启docker

```sh
systemctl daemon-reload
systemctl restart docker
```

## 3.K3S私有镜像仓库配置

需要在每个节点上配置每个节点上配置`/etc/rancher/k3s/registries.yaml`

##### 不使用 TLS并且不认证

```yaml
mirrors:
  docker.io:
    endpoint:
      - "http://mycustomreg.com:5000"
```