---
title: 树莓派EdgeX+TensorflowLite实现机器学习推理
date: 2021-12-27
categories:
- EdgeX
tags:
- EdgeX机器学习以及推理
language: zh-CN
toc: true
---

### 安装 TensorFlow Lite 解释器

需要根据平台以及python版本选择安装，下面是linux(arm64)平台的，python版本为3.7

```bash
pip3 install https://dl.google.com/coral/python/tflite_runtime-2.1.0.post1-cp37-cp37m-linux_aarch64.whl
```

具体参考https://tensorflow.google.cn/lite/guide/python

<!--more-->

### 使用 tflite_runtime 运行推理

```python
#监听8888端口，对收到的图片base64编码解析后进行推理
from __future__ import absolute_import
from __future__ import division
from __future__ import print_function

import base64
import io
import json
import jsonpath
import numpy as np
import time
import argparse

from PIL import Image
from tflite_runtime.interpreter import Interpreter
from flask import Flask,request

app = Flask(__name__)

#测试
@app.route("/hello/", methods=['POST', 'GET'])
def hello():
    return "hello"

#监听并且推理
@app.route("/flask_api/", methods=['POST', 'GET'])
def flask_api():
    parser = argparse.ArgumentParser(
        formatter_class=argparse.ArgumentDefaultsHelpFormatter)
    parser.add_argument(
        '--model', default='lite-model_imagenet_mobilenet_v3_large_075_224_classification_5_metadata_1.tflite',
        required=False)
    args = parser.parse_args()
    interpreter = Interpreter(args.model)
    interpreter.allocate_tensors()
    _, height, width, _ = interpreter.get_input_details()[0]['shape']
    # 获取请求数据
    data2 = json.loads(request.data)
    res = jsonpath.jsonpath(data2, '$..binaryValue')
    res = "".join(res)
    
    # 解析图片数据
    imgdata = base64.b64decode(res)
    input_image = Image.open(io.BytesIO(imgdata))
    input_image = np.array(Image.open(io.BytesIO(imgdata)).resize([width, height]), dtype=np.float32)
    input_image = np.expand_dims(input_image / 255.0, 0)
    
    # 计算推理延迟时间
    start_time = time.time()
    results = classify_image(interpreter, input_image)
    elapsed_ms = (time.time() - start_time)
    
    # 加载标签数据
    strlabel = load_labels('labels.txt')
    result = 'Prediction class: {}, avg latency: {} ms'.format(
        strlabel[results], (elapsed_ms * 1000) )
    
	# 打印推理结果
    print(result)
	
    # 返回推理结果
    return str(results)


def load_labels(path):
    with open(path, 'r') as f:
        return {i: line.strip() for i, line in enumerate(f.readlines())}

    
def set_input_tensor(interpreter, image):
    interpreter.set_tensor(interpreter.get_input_details()[0]["index"], image)


def classify_image(interpreter, image, top_k=1):
    """Returns a sorted array of classification results."""
    set_input_tensor(interpreter, image)
    interpreter.invoke()
    result = interpreter.tensor(interpreter.get_output_details()[0]["index"])()
    return np.argmax(result)


if __name__ == '__main__':
    app.run('0.0.0.0', 8888)

```

### 构建机器学习推理环境镜像

`Dockerfile`

```dockerfile
#基于的基础镜像
FROM python:3.7-slim
# 设置code文件夹是工作目录
WORKDIR /code
#代码添加到code文件夹
COPY .  .
# 安装依赖环境
RUN pip install -r requirements.txt
RUN pip3 install https://dl.google.com/coral/python/tflite_runtime-2.1.0.post1-cp37-cp37m-linux_aarch64.whl
#暴露端口 也是flask监听端口
EXPOSE 8888
#运行python项目
CMD ["python", "app.py"]
```

### 构建EdgeX-AppService镜像

`Dockerfile`

```dockerfile
#
# Copyright (c) 2021 Intel Corporation
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#      http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#

FROM golang:1.16-alpine AS builder

LABEL license='SPDX-License-Identifier: Apache-2.0' \
  copyright='Copyright (c) 2019: Intel'

# add git for go modules
RUN apk update && apk add --no-cache make git gcc libc-dev libsodium-dev zeromq-dev
WORKDIR /post-and-cmd

COPY go.mod .

RUN go env -w GO111MODULE=on
RUN go env -w GOPROXY=https://goproxy.cn,direct

RUN go mod download

COPY . .
RUN apk info -a zeromq-dev

RUN make build

# Next image - Copy built Go binary into new workspace
FROM alpine

LABEL license='SPDX-License-Identifier: Apache-2.0' \
  copyright='Copyright (c) 2019: Intel'

RUN apk --no-cache add zeromq

# Turn off secure mode for examples. Not recommended for production
ENV EDGEX_SECURITY_SECRET_STORE=false

COPY --from=builder /post-and-cmd/res /res
COPY --from=builder /post-and-cmd/app-service /post-and-cmd

CMD [ "/post-and-cmd", "-cp=consul.http://edgex-core-consul:8500", "--registry","--confdir=/res"]
```

`makefile`

```makefile
.PHONY: build clean

GO=CGO_ENABLED=1 GO111MODULE=on go

build:
	go mod tidy
	$(GO) build -o app-service
	
docker:
	docker build \
	--build-arg http_proxy \
	--build-arg https_proxy \
	-f Dockerfile \
	-t caixindi/edgex-app-service-simple-image-classification-http:3.0.1 \
    --platform linux/arm64 \
    --push \
	.
clean:
	rm -f app-service
```

`make docker`执行镜像构建任务

### 树莓派部署(使用K3S)

`.yaml`文件，以下只是新增内容

```yaml
...
  - apiVersion: v1
    kind: Service
    metadata:
      name: tf-listener
    spec:
      type: NodePort
      selector:
        app: tf-listener
      ports:
      - name: tcp-8888
        port: 8888
        protocol: TCP
        targetPort: 8888
        nodePort: 30888
        
  - apiVersion: v1
    kind: Service
    metadata:
      name: edgex-app-service-simple-image-classification-http
    spec:
      type: NodePort
      selector:
        app: edgex-app-service-simple-image-classification-http
      ports:
      - name: tcp-58780
        port: 58786
        protocol: TCP
        targetPort: 58786
...
  - apiVersion: apps/v1
    kind: Deployment
    metadata: 
      name: edgex-app-service-simple-image-classification-http
    spec:
      selector:
        matchLabels: 
          app: edgex-app-service-simple-image-classification-http
      template:
        metadata:
          labels: 
            app: edgex-app-service-simple-image-classification-http
        spec:
          hostname: edgex-app-service-simple-image-classification-http
          containers:
          - name: edgex-app-service-simple-image-classification-http
            image: caixindi/edgex-app-service-simple-image-classification-http:3.0
            imagePullPolicy: IfNotPresent
            ports:
            - name: http
              protocol: TCP
              containerPort: 58786
            env: 
            - name: SERVICE_HOST
              value: "edgex-app-service-simple-image-classification-http"
            - name: TRIGGER_EDGEXMESSAGEBUS_PUBLISHHOST_HOST
              value: "edgex-redis"
            - name: TRIGGER_EDGEXMESSAGEBUS_SUBSCRIBEHOST_HOST
              value: "edgex-redis"
            - name: DATABASE_HOST
              value: "edgex-redis"
            - name: SREVICE_SERVERBINDADDR
              value: "0.0.0.0"
            - name: CLIENTS_CORE_METADATA_HOST
              value: "edgex-core-metadata"
            - name: REGISTRY_HOST
              value: "edgex-core-consul"
            - name: APPLICATIONSETTINGS_HTTPEXPORTURL
              value: "http://192.168.137.100:30888/object detection"
            - name: TRIGGER_EDGEXMESSAGEBUS_SUBSCRIBEHOST_SUBSCRIBETOPICS
              value: "edgex/events/#"
            - name: APPLICATIONSETTINGS_DEVICENAMES
              value: "sample-image"
            - name: CLIENTS_CORE_COMMAND_HOST
              value: "edgex-core-command"
              
  - apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: tf-listener
    spec:
      replicas: 3
      selector:
        matchLabels:
          app: tf-listener
      template:
        metadata:
          labels:
            app: tf-listener
        spec:
          containers:
          - image: caixindi/tf-listener:3.0.0
            name: tf-listener
            imagePullPolicy: IfNotPresent
            ports:
            - name: http
              containerPort: 8888
              protocol: TCP
```

> 部署

`kubectl apply -f ...`

> 发送图片

![](../img/%E6%A0%91%E8%8E%93%E6%B4%BEEdgeX+TensorflowLite%E5%AE%9E%E7%8E%B0%E6%9C%BA%E5%99%A8%E5%AD%A6%E4%B9%A0%E6%8E%A8%E7%90%86/image-20211019191750991.png)

> 查看appservice log

![](../img/%E6%A0%91%E8%8E%93%E6%B4%BEEdgeX+TensorflowLite%E5%AE%9E%E7%8E%B0%E6%9C%BA%E5%99%A8%E5%AD%A6%E4%B9%A0%E6%8E%A8%E7%90%86/image-20211019191722823.png)

> 查看高温预警值

<img src="../img/%E6%A0%91%E8%8E%93%E6%B4%BEEdgeX+TensorflowLite%E5%AE%9E%E7%8E%B0%E6%9C%BA%E5%99%A8%E5%AD%A6%E4%B9%A0%E6%8E%A8%E7%90%86/image-20211019192239889.png" style="zoom: 25%;" />

数值208（从0开始）代表第209类，识别结果为golden retriever.

> 整个项目的逻辑是（虽然并没有意义）：appservice设置触发器，当有图片数据传入时，触发器触发响应事件，向推理程序的监听端口发送图片数据，程序收到图片数据后进行推理，返回推理结果，appservice再将推理结果，也就是最终推理的是第几类别，通过给设备发送命令的方式，将温湿度传感器设置成相应的数值。



### 附：python生成requirements.txt的两种方法

#### 第一种 适用于单虚拟环境的情况：

```powershell
pip freeze > requirements.txt
```

这种方式，会将环境中的依赖包全都加入，如果使用的全局环境，则下载的所有包都会在里面，不管是不是当前项目依赖的。

当然这种情况并不是我们想要的，当我们使用的是全局环境时，可以使用第二种方法。

#### 第二种 使用pipreqs

> 安装

```powershell
pip install pipreqs
```

> 在当前目录执行

```powershell
pipreqs . --encoding=utf8 --force
```

`--encoding=utf8`表示使用 `utf8` 编码，不然可能会报` UnicodeDecodeError: ‘gbk’ codec can’t decode byte 0xae in position 406: illegal multibyte sequence` 的错误。
`--force` 强制执行，当生成目录下的 `requirements.txt `存在时覆盖。

使用 `requirements.txt `安装依赖的方式：

```powershell
pip install -r requirements.txt
```

