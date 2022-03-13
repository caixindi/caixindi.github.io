dockerfile

```
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
WORKDIR /simple-filter-xml-http

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

COPY --from=builder /simple-filter-xml-http/res /res
COPY --from=builder /simple-filter-xml-http/app-service /simple-filter-xml-http

CMD [ "/simple-filter-xml-http", "-cp=consul.http://edgex-core-consul:8500", "--registry","--confdir=/res"]
```

configuration.toml

```toml
[Writable]
LogLevel = "INFO"
  [Writable.StoreAndForward]
  Enabled = false
  RetryInterval = "5m"
  MaxRetryCount = 10
  [Writable.InsecureSecrets]
    [Writable.InsecureSecrets.DB]
    path = "redisdb"
      [Writable.InsecureSecrets.DB.Secrets]
      username = ""
      password = ""

[Service]
HealthCheckInterval = "10s"
Host = "localhost"
Port = 58780 # Adjust if running multiple examples at the same time to avoid duplicate port conflicts
ServerBindAddr = "" # if blank, uses default Go behavior https://golang.org/pkg/net/#Listen
StartupMsg = "This is a sample Filter/XML/HTTP Transform Application Service"
RequestTimeout = "30s"
MaxRequestSize = 0
MaxResultCount = 0

[Registry]
Host = "localhost"
Port = 8500
Type = "consul"

# Database is require when Store and Forward is enabled
[Database]
Type = "redisdb"
Host = "localhost"
Port = 6379
Timeout = "30s"

# SecretStore is required when Store and Forward is enabled and running with security
# so Database credentials can be pulled from Vault.
# Note when running in docker from compose file set the following environment variables:
#   - SecretStore_Host: edgex-vault
[SecretStore]
Type = "vault"
Host = "localhost"
Port = 8200
Path = "app-simple-filter-xml-http/"
Protocol = "http"
TokenFile = "/tmp/edgex/secrets/app-simple-filter-xml-http/secrets-token.json"
RootCaCertPath = ""
ServerName = ""
  [SecretStore.Authentication]
  AuthType = "X-Vault-Token"

[Clients]
  [Clients.core-metadata]
  Protocol = "http"
  Host = "localhost"
  Port = 59881

# Choose either an HTTP trigger or edgex-messagebus trigger

#[Trigger]
#Type="http"

[Trigger]
Type="edgex-messagebus"
  [Trigger.EdgexMessageBus]
  Type = "redis"
    [Trigger.EdgexMessageBus.SubscribeHost]
    Host = "localhost"
    Port = 6379
    Protocol = "redis"
    SubscribeTopics="edgex/events/#"
    [Trigger.EdgexMessageBus.PublishHost]
    Host = "localhost"
    Port = 6379
    Protocol = "redis"
    PublishTopic="example"

# App Service specifc simple settings
# Great for single string settings. For more complex structured custom configuration
# See https://docs.edgexfoundry.org/2.0/microservices/application/AdvancedTopics/#custom-configuration
[ApplicationSettings]
DeviceNames = "Random-Float-Device, Random-Integer-Device"
HttpExportUrl = "http://0.0.0.0:8081/index" # TODO: Replace with your endpoint
```

makeflie

```
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
	-t edgexfoundry/docker-simple-filter-xml-http:1.0 \
	--platform linux/arm64 \
	.
clean:
	rm -f app-service
```

main.go

```go
package main

import (
	"encoding/xml"
	"fmt"
	"io"
	"io/ioutil"
	"log"
	"net/http"
	"github.com/gin-gonic/gin"
)

type event struct {
	XMLName xml.Name `xml:"Event"`
	ApiVersion string `xml:"ApiVersion"`
	Id string `xml:"Id"`
	DeviceName string `xml:"DeviceName"`
	ProfileName string `xml:"ProfileName"`
	SourceName string `xml:"SourceName"`
	Origin string `xml:"Origin"`
	Readings readings `xml:"Readings"`
}

type readings struct {
	Id string `xml:"Id"`
	Origin string `xml:"Origin"`
	DeviceName string `xml:"DeviceName"`
	ResourceName string `xml:"ResourceName"`
	ProfileName string `xml:"ProfileName"`
	ValueType string `xml:"ValueType"`
	BinaryValue string `xml:"BinaryValue"`
	MediaType string `xml:"MediaType"`
	Value string `xml:"Value"`
}

func IndexHandler(w http.ResponseWriter, r *http.Request)  {
	//打印请求主机地址
	//fmt.Println(r.Host)
	//打印请求头信息
	//fmt.Printf("header content:[%v]\n", r.Header)
	//获取 post 请求中 form 里边的数据
	//fmt.Printf("form content:[%s, %s]\n", r.PostFormValue("username"), r.PostFormValue("passwd"))
	//读取请求体信息
	bodyContent, err := ioutil.ReadAll(r.Body)
	if err != nil && err != io.EOF {
		fmt.Printf("read body content failed, err:[%s]\n", err.Error())
		return
	}
	//fmt.Println("body content:[%s]\n", string(bodyContent))
	v:= event{}
	err = xml.Unmarshal(bodyContent, &v)
	//返回响应内容
	if v.Readings.DeviceName == "" {
		fmt.Fprintf(w, "Hi,get method rev")
		fmt.Println("get method rev")
	}else{
		fmt.Fprintf(w, "Data accepted successfully")
		fmt.Println("设备:"+v.Readings.DeviceName+"  value:"+v.Readings.Value)
	}

}

func main()  {
	http.HandleFunc("/", IndexHandler)
	err := http.ListenAndServe(":8999",nil)
	if err != nil {
		log.Printf(err.Error())
	} else {
		log.Printf("成功监听")
	}
}
```

