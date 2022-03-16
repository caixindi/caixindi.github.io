---
title: device-sdk-go源码分析
date: 2021-12-27
categories:
- EdgeX
tags:
- EdgeX概念
language: zh-CN
toc: true
---

### device-sdk-go源码分析

#### device-sdk-go项目目录结构

```
├─.github
│  └─ISSUE_TEMPLATE
├─bin
├─example
│  ├─cmd
│  │  └─device-simple
│  │      └─res
│  │          ├─devices
│  │          └─profiles
│  ├─config
│  └─driver
├─internal
│  ├─application
│  ├─autodiscovery
│  ├─autoevent
│  ├─cache
│  ├─clients
│  ├─common
│  ├─config
│  ├─container
│  ├─controller
│  │  └─http
│  │      └─correlation
│  ├─messaging
│  ├─provision
│  ├─telemetry
│  └─transformer
├─openapi
│  └─v2
├─pkg
│  ├─models
│  │  └─mocks
│  ├─service
│  └─startup
└─snap
    ├─hooks
    └─local

```

<!--more-->

接下来以官方提供的example项目包为例，分析device-sdk-go的项目结构和相关功能。

目录结构如下：

```
├─cmd
│  └─device-simple
│      └─res
│          ├─devices
│          └─profiles
├─config
└─driver

```

##### cmd文件夹

`main.go`

```go
// -*- Mode: Go; indent-tabs-mode: t -*-
//
// Copyright (C) 2017-2018 Canonical Ltd
// Copyright (C) 2018-2019 IOTech Ltd
//
// SPDX-License-Identifier: Apache-2.0

// This package provides a simple example of a device service.
package main
//导入包
import (
	"github.com/edgexfoundry/device-sdk-go/v2"
	"github.com/edgexfoundry/device-sdk-go/v2/example/driver"
	"github.com/edgexfoundry/device-sdk-go/v2/pkg/startup"
)

//定义常量 服务名称
const (
	serviceName string = "device-simple"
)

func main() {
	sd := driver.SimpleDriver{}  //初始化SimpleDriver结构体
	startup.Bootstrap(serviceName, device.Version, &sd)  //通过引导程序启动设备服务
}

```

`configuration.toml`

- 项目配置文件
- 描述当前device-service的ip，port以及所依赖的各个EdgeX微服务的配置信息等

```toml
[Writable]
LogLevel = "INFO"
  # 非安全模式下的配置
  # Example InsecureSecrets configuration that simulates SecretStore for when EDGEX_SECURITY_SECRET_STORE=false
  # InsecureSecrets are required for when Redis is used for message bus
  [Writable.InsecureSecrets]
    [Writable.InsecureSecrets.DB]
    path = "redisdb"
      [Writable.InsecureSecrets.DB.Secrets]
      username = ""
      password = ""

[Service]
HealthCheckInterval = "10s" #健康健康间隔
Host = "localhost" 
Port = 59999 # Device serivce are assigned the 599xx range
ServerBindAddr = ""  # blank value defaults to Service.Host value
StartupMsg = "device simple started"
# MaxRequestSize limit the request body size in byte of put command
MaxRequestSize = 0 # value 0 unlimit the request size.
RequestTimeout = "20s"
  [Service.CORSConfiguration]
  EnableCORS = false
  CORSAllowCredentials = false
  CORSAllowedOrigin = "https://localhost"
  CORSAllowedMethods = "GET, POST, PUT, PATCH, DELETE"
  CORSAllowedHeaders = "Authorization, Accept, Accept-Language, Content-Language, Content-Type, X-Correlation-ID"
  CORSExposeHeaders = "Cache-Control, Content-Language, Content-Length, Content-Type, Expires, Last-Modified, Pragma, X-Correlation-ID"
  CORSMaxAge = 3600

[Registry]
Host = "localhost"
Port = 8500
Type = "consul"

[Clients]
  [Clients.core-data]
  Protocol = "http"
  Host = "localhost"
  Port = 59880

  [Clients.core-metadata]
  Protocol = "http"
  Host = "localhost"
  Port = 59881

[MessageQueue]
Protocol = "redis"
Host = "localhost"
Port = 6379
Type = "redis"
AuthMode = "usernamepassword"  # required for redis messagebus (secure or insecure).
SecretName = "redisdb"
PublishTopicPrefix = "edgex/events/device" # /<device-profile-name>/<device-name>/<source-name> will be added to this Publish Topic prefix
  [MessageQueue.Optional]
  # Default MQTT Specific options that need to be here to enable environment variable overrides of them
  # Client Identifiers
  ClientId = "device-simple"
  # Connection information
  Qos = "0" # Quality of Sevice values are 0 (At most once), 1 (At least once) or 2 (Exactly once)
  KeepAlive = "10" # Seconds (must be 2 or greater)
  Retained = "false"
  AutoReconnect = "true"
  ConnectTimeout = "5" # Seconds
  SkipCertVerify = "false" # Only used if Cert/Key file or Cert/Key PEMblock are specified

# Example SecretStore configuration.
# Only used when EDGEX_SECURITY_SECRET_STORE=true
# Must also add `ADD_SECRETSTORE_TOKENS: "device-simple"` to vault-worker environment so it generates
# the token and secret store in vault for "device-simple"
[SecretStore]
Type = "vault"
Host = "localhost"
Port = 8200
Path = "device-simple/"
Protocol = "http"
RootCaCertPath = ""
ServerName = ""
SecretsFile = ""
DisableScrubSecretsFile = false
TokenFile = "/tmp/edgex/secrets/device-simple/secrets-token.json"
  [SecretStore.Authentication]
  AuthType = "X-Vault-Token"

[Device]
  DataTransform = true
  MaxCmdOps = 128
  MaxCmdValueLen = 256
  ProfilesDir = "./res/profiles"
  DevicesDir = "./res/devices"
  UpdateLastConnected = false
  AsyncBufferSize = 1
  EnableAsyncReadings = true
  Labels = []
  UseMessageBus = true
  [Device.Discovery]
    Enabled = false
    Interval = "30s"

# Example structured custom configuration
[SimpleCustom]
OnImageLocation = "./res/on.png"
OffImageLocation = "./res/off.jpg"
  [SimpleCustom.Writable]
  DiscoverSleepDurationSecs = 10

```

##### config文件夹

`configuration.go`

```go
package config

import (
	"errors"
)
//此文件包含了可以从服务的configuration.toml加载的配置示例
//或者是配置提供程序，又名consul

// 如果要在configuration.toml中使用自定义配置类型
//那么在configuration.toml的顶级配置名须是本文件中最外部结构名
//比如这个示例的 SimpleCustom
type ServiceConfig struct {
	SimpleCustom SimpleCustomConfig
}

type SimpleCustomConfig struct {
	OffImageLocation string
	OnImageLocation  string
	Writable         SimpleWritable
}

type SimpleWritable struct {
	DiscoverSleepDurationSecs int64
}

// 从接受到的原始数据更新服务的完整配置
func (sw *ServiceConfig) UpdateFromRaw(rawConfig interface{}) bool {
	configuration, ok := rawConfig.(*ServiceConfig)
	if !ok {
		return false //errors.New("unable to cast raw config to type 'ServiceConfig'")
	}

	*sw = *configuration

	return true
}

// 确保自定义配置都有正确的值
func (scc *SimpleCustomConfig) Validate() error {
	if len(scc.OnImageLocation) == 0 {
		return errors.New("SimpleCustom.OnImageLocation configuration setting can not be blank")
	}

	if len(scc.OffImageLocation) == 0 {
		return errors.New("SimpleCustom.OffImageLocation configuration setting can not be blank")
	}

	if scc.Writable.DiscoverSleepDurationSecs < 10 {
		return errors.New("SimpleCustom.Writable.DiscoverSleepDurationSecs configuration setting must be 10 or greater")
	}

	return nil
}

```

driver文件夹

`simpledriver.go`

```go
//这个包提供了协议驱动接口的简单实现示例
package driver
//导入包
import (
	"bytes"
	"fmt"
	"image"
	"image/jpeg"
	"image/png"
	"os"
	"reflect"
	"time"

	"github.com/edgexfoundry/go-mod-core-contracts/v2/clients/logger"
	"github.com/edgexfoundry/go-mod-core-contracts/v2/common"
	"github.com/edgexfoundry/go-mod-core-contracts/v2/models"

	"github.com/edgexfoundry/device-sdk-go/v2/example/config"
	sdkModels "github.com/edgexfoundry/device-sdk-go/v2/pkg/models"
	"github.com/edgexfoundry/device-sdk-go/v2/pkg/service"
)
//简单驱动结构
type SimpleDriver struct {
	lc            logger.LoggingClient   //日志客户端
	asyncCh       chan<- *sdkModels.AsyncValues  //用于通过 ProtocolDrivers 异步发送设备读数的结构
	deviceCh      chan<- []sdkModels.DiscoveredDevice //定义找到一个设备所需的信息
	switchButton  bool //开关按钮
	xRotation     int32 //x轴旋转速度
	yRotation     int32 //y轴旋转速度
	zRotation     int32 //z轴旋转速度
	counter       interface{} //计数器
	serviceConfig *config.ServiceConfig  //自定义配置
}

//获取图片
func getImageBytes(imgFile string, buf *bytes.Buffer) error {
	// 从文件中读取图片
	img, err := os.Open(imgFile)
	if err != nil {
		return err
	}
	defer img.Close()

	// 解码，判断图片类型
	imageData, imageType, err := image.Decode(img)
	if err != nil {
		return err
	}
	// 当前文件任务结束，重置文件pointer
	img.Seek(0, 0)
	if imageType == "jpeg" {
		err = jpeg.Encode(buf, imageData, nil)
		if err != nil {
			return err
		}
	} else if imageType == "png" {
		err = png.Encode(buf, imageData)
		if err != nil {
			return err
		}
	}
	return nil
}

// 为设备服务执行特定于协议的初始化
func (s *SimpleDriver) Initialize(lc logger.LoggingClient, asyncCh chan<- *sdkModels.AsyncValues, deviceCh chan<- []sdkModels.DiscoveredDevice) error {
    //为每个成员赋值
	s.lc = lc
	s.asyncCh = asyncCh
	s.deviceCh = deviceCh
    //自定义服务配置
	s.serviceConfig = &config.ServiceConfig{}
    //计数器内容为map结构 key类型为string value类型为interface{}
	s.counter = map[string]interface{}{
		"f1": "ABC",
		"f2": 123,
	}
	
    //service.RunningService() 返回设备服务
    //在"github.com/edgexfoundry/device-sdk-go/v2/pkg/service"中
    //定义了    var(
    //          ds *DeviceService
    //          )
	ds := service.RunningService()
	
    //加载自定义服务配置
	if err := ds.LoadCustomConfig(s.serviceConfig, "SimpleCustom"); err != nil {
		return fmt.Errorf("unable to load 'SimpleCustom' custom configuration: %s", err.Error())
	}
	
    //输出当前自定义配置
	lc.Infof("Custom config is: %v", s.serviceConfig.SimpleCustom)
    
	//确保自定义配置具有正确的值
	if err := s.serviceConfig.SimpleCustom.Validate(); err != nil {
		return fmt.Errorf("'SimpleCustom' custom configuration validation failed: %s", err.Error())
	}
	
    //启用go-mod-bootstrap中的配置处理器监听对自定义配置部分的更改
    //注意LoadCustomConfig必须在此方法前调用，因为在LoadCustomConfig方法内生成了configProcessor实例
    //所需传入的三个参数分别为需要监听的配置内容(可写的),名称,回调函数
	if err := ds.ListenForCustomConfigChanges(
		&s.serviceConfig.SimpleCustom.Writable,
		"SimpleCustom/Writable", s.ProcessCustomConfigChanges); err != nil {
		return fmt.Errorf("unable to listen for changes for 'SimpleCustom.Writable' custom configuration: %s", err.Error())
	}

	return nil
}

// 处理自定义配置更改
func (s *SimpleDriver) ProcessCustomConfigChanges(rawWritableConfig interface{}) {
    //类型断言
	updated, ok := rawWritableConfig.(*config.SimpleWritable)
	if !ok {
		s.lc.Error("unable to process custom config updates: Can not cast raw config to type 'SimpleWritable'")
		return
	}

	s.lc.Info("Received configuration updates for 'SimpleCustom.Writable' section")

	previous := s.serviceConfig.SimpleCustom.Writable
	s.serviceConfig.SimpleCustom.Writable = *updated

    //判断是否有改动
	if reflect.DeepEqual(previous, *updated) {
		s.lc.Info("No changes detected")
		return
	}

    // 检查发生了什么变化,以下只是一个示例，并不适合于所有情况
	if previous.DiscoverSleepDurationSecs != updated.DiscoverSleepDurationSecs {
		s.lc.Infof("DiscoverSleepDurationSecs changed to: %d", updated.DiscoverSleepDurationSecs)
	}
}

// HandleReadCommands triggers a protocol Read operation for the specified device.
// 触发指定设备的协议读取操作
// 所需传入的三个参数分别为: 设备名称、设备连接信息、请求的命令内容、命令参数值
func (s *SimpleDriver) HandleReadCommands(deviceName string, protocols map[string]models.ProtocolProperties, reqs []sdkModels.CommandRequest) (res []*sdkModels.CommandValue, err error) {
	s.lc.Debugf("SimpleDriver.HandleReadCommands: protocols: %v resource: %v attributes: %v", protocols, reqs[0].DeviceResourceName, reqs[0].Attributes)

	if len(reqs) == 1 {
		res = make([]*sdkModels.CommandValue, 1)
		if reqs[0].DeviceResourceName == "SwitchButton" {
            // 根据valueType创建相关命令
			cv, _ := sdkModels.NewCommandValue(reqs[0].DeviceResourceName, common.ValueTypeBool, s.switchButton)
			res[0] = cv
		} else if reqs[0].DeviceResourceName == "Xrotation" {
			cv, _ := sdkModels.NewCommandValue(reqs[0].DeviceResourceName, common.ValueTypeInt32, s.xRotation)
			res[0] = cv
		} else if reqs[0].DeviceResourceName == "Yrotation" {
			cv, _ := sdkModels.NewCommandValue(reqs[0].DeviceResourceName, common.ValueTypeInt32, s.yRotation)
			res[0] = cv
		} else if reqs[0].DeviceResourceName == "Zrotation" {
			cv, _ := sdkModels.NewCommandValue(reqs[0].DeviceResourceName, common.ValueTypeInt32, s.zRotation)
			res[0] = cv
		} else if reqs[0].DeviceResourceName == "Image" {
			// 显示开关的二进制图像表示
			buf := new(bytes.Buffer)
			if s.switchButton == true {
				err = getImageBytes(s.serviceConfig.SimpleCustom.OnImageLocation, buf)
			} else {
				err = getImageBytes(s.serviceConfig.SimpleCustom.OffImageLocation, buf)
			}
			cvb, _ := sdkModels.NewCommandValue(reqs[0].DeviceResourceName, common.ValueTypeBinary, buf.Bytes())
			res[0] = cvb
		} else if reqs[0].DeviceResourceName == "Uint8Array" {
			cv, _ := sdkModels.NewCommandValue(reqs[0].DeviceResourceName, common.ValueTypeUint8Array, []uint8{0, 1, 2})
			res[0] = cv
		} else if reqs[0].DeviceResourceName == "Counter" {
			cv, _ := sdkModels.NewCommandValue(reqs[0].DeviceResourceName, common.ValueTypeObject, s.counter)
			res[0] = cv
		}
	} else if len(reqs) == 3 {
		res = make([]*sdkModels.CommandValue, 3)
		for i, r := range reqs {
			var cv *sdkModels.CommandValue
			switch r.DeviceResourceName {
			case "Xrotation":
				cv, _ = sdkModels.NewCommandValue(r.DeviceResourceName, common.ValueTypeInt32, s.xRotation)
			case "Yrotation":
				cv, _ = sdkModels.NewCommandValue(r.DeviceResourceName, common.ValueTypeInt32, s.yRotation)
			case "Zrotation":
				cv, _ = sdkModels.NewCommandValue(r.DeviceResourceName, common.ValueTypeInt32, s.zRotation)
			}
			res[i] = cv
		}
	}

	return
}

// 传递一个CommandRequest结构的切片,每个切片代表对特定设备资源的操作。由于这些命令是驱动命令,参数为单个命令提供参数
// 所需传入的三个参数分别为: 设备名称、设备连接信息、请求的命令内容
func (s *SimpleDriver) HandleWriteCommands(deviceName string, protocols map[string]models.ProtocolProperties, reqs []sdkModels.CommandRequest,
	params []*sdkModels.CommandValue) error {
	var err error

	for i, r := range reqs {
		s.lc.Debugf("SimpleDriver.HandleWriteCommands: protocols: %v, resource: %v, parameters: %v, attributes: %v", protocols, reqs[i].DeviceResourceName, params[i], reqs[i].Attributes)
		switch r.DeviceResourceName {
		case "SwitchButton":
			if s.switchButton, err = params[i].BoolValue(); err != nil {
				err := fmt.Errorf("SimpleDriver.HandleWriteCommands; the data type of parameter should be Boolean, parameter: %s", params[0].String())
				return err
			}
		case "Xrotation":
			if s.xRotation, err = params[i].Int32Value(); err != nil {
				err := fmt.Errorf("SimpleDriver.HandleWriteCommands; the data type of parameter should be Int32, parameter: %s", params[i].String())
				return err
			}
		case "Yrotation":
			if s.yRotation, err = params[i].Int32Value(); err != nil {
				err := fmt.Errorf("SimpleDriver.HandleWriteCommands; the data type of parameter should be Int32, parameter: %s", params[i].String())
				return err
			}
		case "Zrotation":
			if s.zRotation, err = params[i].Int32Value(); err != nil {
				err := fmt.Errorf("SimpleDriver.HandleWriteCommands; the data type of parameter should be Int32, parameter: %s", params[i].String())
				return err
			}
		case "Uint8Array":
			v, err := params[i].Uint8ArrayValue()
			if err == nil {
				s.lc.Debugf("Uint8 array value from write command: ", v)
			} else {
				return err
			}
		case "Counter":
			if s.counter, err = params[i].ObjectValue(); err != nil {
				err := fmt.Errorf("SimpleDriver.HandleWriteCommands; the data type of parameter should be Object, parameter: %s", params[i].String())
				return err
			}
		}
	}

	return nil
}

// 协议特定的设备服务代码将正常关闭，或者如果force参数为“true”，则立即关闭。
// 驱动程序负责关闭任何正在使用的通道，包括用于发送异步读数的通道(如果支持)。
func (s *SimpleDriver) Stop(force bool) error {
	// Then Logging Client might not be initialized
	if s.lc != nil {
		s.lc.Debugf("SimpleDriver.Stop called: force=%v", force)
	}
	return nil
}

// AddDevice是一个回调函数，在添加与此设备服务关联的新设备时调用.
func (s *SimpleDriver) AddDevice(deviceName string, protocols map[string]models.ProtocolProperties, adminState models.AdminState) error {
	s.lc.Debugf("a new Device is added: %s", deviceName)
	return nil
}

// UpdateDevice是一个回调函数，在更新与此设备服务关联的设备时调用
func (s *SimpleDriver) UpdateDevice(deviceName string, protocols map[string]models.ProtocolProperties, adminState models.AdminState) error {
	s.lc.Debugf("Device %s is updated", deviceName)
	return nil
}

// RemoveDevice是一个回调函数，在删除与此设备服务关联的设备时调用
func (s *SimpleDriver) RemoveDevice(deviceName string, protocols map[string]models.ProtocolProperties) error {
	s.lc.Debugf("Device %s is removed", deviceName)
	return nil
}

// 用于发现特定协议的设备，这是一种异步操作
// 在发现过程中如果找到设备将写入设备操作 
func (s *SimpleDriver) Discover() {
	proto := make(map[string]models.ProtocolProperties)
	proto["other"] = map[string]string{"Address": "simple02", "Port": "301"}

	device2 := sdkModels.DiscoveredDevice{
		Name:        "Simple-Device02",
		Protocols:   proto,
		Description: "found by discovery",
		Labels:      []string{"auto-discovery"},
	}

	proto = make(map[string]models.ProtocolProperties)
	proto["other"] = map[string]string{"Address": "simple03", "Port": "399"}

	device3 := sdkModels.DiscoveredDevice{
		Name:        "Simple-Device03",
		Protocols:   proto,
		Description: "found by discovery",
		Labels:      []string{"auto-discovery"},
	}

	res := []sdkModels.DiscoveredDevice{device2, device3}

	time.Sleep(time.Duration(s.serviceConfig.SimpleCustom.Writable.DiscoverSleepDurationSecs) * time.Second)
	s.deviceCh <- res
}

```



`container.go` 采用了控制反转的设计模式(IOC)，实现了一个简单的依赖注入容器。为什么要这么做？

　●控制：传统的面向对象程序设计，如果我们需要控制对象，可以直接在对象内部通过new进行创建对象，是程序主动去创建依赖对象；而IOC则是专门有一个容器去创建这些对象，即通过IOC容器来控制对象的创建。所以这边就不是原本的程序控制了对象，而是IOC容器控制了对象；控制了什么？控制了程序的外部资源获取，也就是说现在应用程序需要获取外部资源，那么就需要通过容器来进行操作，显然这降低了程序间的耦合性。

   ●反转：在没有容器的时候，我们需要一个对象的时候，我们必须自己new一个对象，这时候主动权在自己手上，我们可以称之为正转，而现在我们将控制权交给了容器。所以控制反转IOC就是说将创建对象的控制权进行转移，之前创建对象的主动权和创建时机是由自己把控的，而现在这种权力转移到第三方，这边是容器，它就是一个专门用来创建对象的工厂，你要什么对象，它就给你什么对象，有了 IOC容器，此时的依赖关系就变了，原先可能A依赖B，B依赖C，但是现在他们都是依赖IOC容器。

```go
package di

import "sync"
// 定义一个函数类型
type Get func(serviceName string) interface{}

// 定义一个函数类型
type ServiceConstructor func(get Get) interface{}

// ServiceConstructorMap maps a service name to a function/closure to create that service.
type ServiceConstructorMap map[string]ServiceConstructor

// service is an internal structure used to track a specific service's constructor and constructed instance.
type service struct {
	constructor ServiceConstructor
	instance    interface{}
}

// Container is a receiver that maintains a list of services, their constructors, and their constructed instances in a
// thread-safe manner.
type Container struct {
	serviceMap map[string]service
	mutex      sync.RWMutex
}

// NewContainer是一个工厂方法，返回初始化的容器接收器
func NewContainer(serviceConstructors ServiceConstructorMap) *Container {
	c := Container{
		serviceMap: map[string]service{},
		mutex:      sync.RWMutex{},
	}
	if serviceConstructors != nil {
		c.Update(serviceConstructors)
	}
	return &c
}

// 使用提供的ServiceConstructorMap的内容更新serviceMap。
func (c *Container) Update(serviceConstructors ServiceConstructorMap) {
	c.mutex.Lock()
	defer c.mutex.Unlock()
	for serviceName, constructor := range serviceConstructors {
		c.serviceMap[serviceName] = service{
			constructor: constructor,
			instance:    nil,
		}
	}
}

// get查找请求的serviceName，如果它存在，则返回一个构造的实例。如果请求的服务不存在，则返回nil。
// Get 获取单个实例中的实例构造，实现如果一个实例已经被构造，将被重用并返回给所有后续的get（serviceName）调用。
func (c *Container) get(serviceName string) interface{} {
	service, ok := c.serviceMap[serviceName]
	if !ok {
		return nil
	}
	if service.instance == nil {
		service.instance = service.constructor(c.get)
		c.serviceMap[serviceName] = service
	}
	return service.instance
}

// 这边实现了container的Get方法，为什么要这么做，主要就是为了用单例模式来保证线程安全。
func (c *Container) Get(serviceName string) interface{} {
	c.mutex.Lock()
	defer c.mutex.Unlock()
	return c.get(serviceName)
}
```



​	SDK中提供了这么一个简单的示例用法:

​	首先定义了两种结构体，并且给出了相应的初始化方法，在main函数中，直接讲

```go

package main

import (
	"fmt"
	"github.com/edgexfoundry/go-mod-bootstrap/v2/di"
)

type foo struct {
	FooMessage string
}

func NewFoo(m string) *foo {
	return &foo{
		FooMessage: m,
	}
}

type bar struct {
	BarMessage string
	Foo        *foo
}

func NewBar(m string, foo *foo) *bar {
	return &bar{
		BarMessage: m,
		Foo:        foo,
	}
}

func main() {
	container := di.NewContainer(
		di.ServiceConstructorMap{
            // 这边传入的函数get没有用到，这边直接返回了一个实例
			"foo": func(get di.Get) interface{} {
				return NewFoo("fooMessage")
			},
			"bar": func(get di.Get) interface{} {
				return NewBar("barMessage", get("foo").(*foo))
			},
		})

	b := container.Get("bar").(*bar)
	fmt.Println(b.BarMessage)
	fmt.Println(b.Foo.FooMessage)
}
```

```go
lc := container.LoggingClientFrom(dic.Get)

// 这个函数方法也是ServiceConstructor类型的
func LoggingClientFrom(get di.Get) logger.LoggingClient {
	loggingClient, ok := get(LoggingClientInterfaceName).(logger.LoggingClient)
	if !ok {
		return nil
	}

	return loggingClient
}
```





command struct

```go
type CommandProcessor struct {
	device        models.Device
	sourceName    string
	correlationID string
	setParamsMap  map[string]interface{}
	attributes    string
	dic           *di.Container
}

type profileCache struct {
	deviceProfileMap  map[string]*models.DeviceProfile // key is DeviceProfile name
	deviceResourceMap map[string]map[string]models.DeviceResource
	deviceCommandMap  map[string]map[string]models.DeviceCommand
	mutex             sync.RWMutex
}

type DeviceResource struct {
	Description string
	Name        string
	IsHidden    bool
	Tag         string
	Properties  ResourceProperties
	Attributes  map[string]interface{}
}


type DeviceCommand struct {
	Name               string
	IsHidden           bool
	ReadWrite          string
	ResourceOperations []ResourceOperation
}

type ResourceOperation struct {
	DeviceResource string
	DefaultValue   string
	Mappings       map[string]string
}


type CommandRequest struct {
	// DeviceResourceName is the name of Device Resource for this command
	DeviceResourceName string
	// Attributes is a key/value map to represent the attributes of the Device Resource
	Attributes map[string]interface{}
	// Type is the data type of the Device Resource
	Type string
}

// CommandValue is the struct to represent the reading value of a Get command coming
// from ProtocolDrivers or the parameter of a Put command sending to ProtocolDrivers.
type CommandValue struct {
	// DeviceResourceName is the name of Device Resource for this command
	DeviceResourceName string
	// Type indicates what type of value was returned from the ProtocolDriver instance in
	// response to HandleCommand being called to handle a single ResourceOperation.
	Type string
	// Value holds value returned by a ProtocolDriver instance.
	// The value can be converted to its native type by referring to ValueType.
	Value interface{}
	// Origin is an int64 value which indicates the time the reading
	// contained in the CommandValue was read by the ProtocolDriver
	// instance.
	Origin int64
	// Tags allows device service to add custom information to the Event in order to
	// help identify its origin or otherwise label it before it is send to north side.
	Tags map[string]string
}
```

`command.go`

```go
// -*- Mode: Go; indent-tabs-mode: t -*-
//
// Copyright (C) 2020-2021 IOTech Ltd
//
// SPDX-License-Identifier: Apache-2.0

package application

import (
	"bytes"
	"encoding/base64"
	"encoding/binary"
	"encoding/json"
	"fmt"
	"math"
	"strconv"
	"strings"

	bootstrapContainer "github.com/edgexfoundry/go-mod-bootstrap/v2/bootstrap/container"
	"github.com/edgexfoundry/go-mod-bootstrap/v2/di"
	"github.com/edgexfoundry/go-mod-core-contracts/v2/common"
	"github.com/edgexfoundry/go-mod-core-contracts/v2/dtos"
	"github.com/edgexfoundry/go-mod-core-contracts/v2/errors"
	"github.com/edgexfoundry/go-mod-core-contracts/v2/models"

	"github.com/edgexfoundry/device-sdk-go/v2/internal/cache"
	sdkCommon "github.com/edgexfoundry/device-sdk-go/v2/internal/common"
	"github.com/edgexfoundry/device-sdk-go/v2/internal/container"
	"github.com/edgexfoundry/device-sdk-go/v2/internal/transformer"
	sdkModels "github.com/edgexfoundry/device-sdk-go/v2/pkg/models"
)

type CommandProcessor struct {
	device        models.Device
	sourceName    string
	correlationID string
	setParamsMap  map[string]interface{}
	attributes    string
	dic           *di.Container
}

func NewCommandProcessor(device models.Device, sourceName string, correlationID string, setParamsMap map[string]interface{}, attributes string, dic *di.Container) *CommandProcessor {
	if setParamsMap == nil {
		setParamsMap = make(map[string]interface{})
	}
	return &CommandProcessor{
		device:        device,
		sourceName:    sourceName,
		correlationID: correlationID,
		setParamsMap:  setParamsMap,
		attributes:    attributes,
		dic:           dic,
	}
}

// CommandHandler 命令处理
func CommandHandler(isRead bool, sendEvent bool, correlationID string, vars map[string]string, setParamsMap map[string]interface{}, attributes string, dic *di.Container) (res *dtos.Event, err errors.EdgeX) {
	// check device service AdminState
	// 检查设备服务管理状态
	ds := container.DeviceServiceFrom(dic.Get)
	// 锁定状态
	if ds.AdminState == models.Locked {
		return res, errors.NewCommonEdgeX(errors.KindServiceLocked, "service locked", nil)
	}

	// check provided device exists
	// 检查设备是否存在
	deviceKey := vars[common.Name]
	device, ok := cache.Devices().ForName(deviceKey)
	if !ok {
		return res, errors.NewCommonEdgeX(errors.KindEntityDoesNotExist, fmt.Sprintf("device %s not found", deviceKey), nil)
	}

	// check device's AdminState
	// 检查设备管理状态
	if device.AdminState == models.Locked {
		return res, errors.NewCommonEdgeX(errors.KindServiceLocked, fmt.Sprintf("device %s locked", device.Name), nil)
	}
	// check device's OperatingState
	// 检查设备操作状态
	if device.OperatingState == models.Down {
		return res, errors.NewCommonEdgeX(errors.KindServiceLocked, fmt.Sprintf("device %s OperatingState is DOWN", device.Name), nil)
	}
	// the device service will perform some operations(e.g. update LastConnected timestamp,
	// push returning event to core-data) after a device is successfully interacted with if
	// it has been configured to do so, and those operation apply to every protocol and
	// need to be finished in the end of application layer before returning to protocol layer.
	// 设备服务将在与设备成功交互后执行一些操作
	// 例如更新LastConnected timestamp(最后一次连接时间戳)、将返回事件推送到核心数据
	// 如果设备已被配置为执行这些操作，则这些操作将应用于每个协议，并且需要在应用层末尾完成，然后再返回到协议层。
	defer func() {
		if err != nil {
			return
		}
		config := container.ConfigurationFrom(dic.Get)
		// 更新最后一次连接时间戳
		if config.Device.UpdateLastConnected {
			go sdkCommon.UpdateLastConnected(device.Name, bootstrapContainer.LoggingClientFrom(dic.Get), bootstrapContainer.MetadataDeviceClientFrom(dic.Get))
		}
		// 将返回时间推送到核心数据
		if res != nil && sendEvent {
			go sdkCommon.SendEvent(res, correlationID, dic)
		}
	}()

	cmd := vars[common.Command]
	// 创建命令处理器对象
	helper := NewCommandProcessor(device, cmd, correlationID, setParamsMap, attributes, dic)
	// 判断命令是否存在
	_, cmdExist := cache.Profiles().DeviceCommand(device.ProfileName, cmd)
	if cmdExist {
		// 如果只读,返回读设备命令
		if isRead {
			return helper.ReadDeviceCommand()
		} else {
			return res, helper.WriteDeviceCommand()
		}
	} else {
		if isRead {
			return helper.ReadDeviceResource()
		} else {
			return res, helper.WriteDeviceResource()
		}
	}
}

func (c *CommandProcessor) ReadDeviceResource() (res *dtos.Event, e errors.EdgeX) {
	dr, ok := cache.Profiles().DeviceResource(c.device.ProfileName, c.sourceName)
	if !ok {
		errMsg := fmt.Sprintf("deviceResource %s not found", c.sourceName)
		return res, errors.NewCommonEdgeX(errors.KindEntityDoesNotExist, errMsg, nil)
	}
	// check deviceResource is not write-only
	if dr.Properties.ReadWrite == common.ReadWrite_W {
		errMsg := fmt.Sprintf("deviceResource %s is marked as write-only", dr.Name)
		return res, errors.NewCommonEdgeX(errors.KindNotAllowed, errMsg, nil)
	}

	lc := bootstrapContainer.LoggingClientFrom(c.dic.Get)
	lc.Debugf("Application - readDeviceResource: reading deviceResource: %s; %s: %s", dr.Name, common.CorrelationHeader, c.correlationID)

	var req sdkModels.CommandRequest
	var reqs []sdkModels.CommandRequest

	// prepare CommandRequest
	req.DeviceResourceName = dr.Name
	req.Attributes = dr.Attributes
	if c.attributes != "" {
		if len(req.Attributes) <= 0 {
			req.Attributes = make(map[string]interface{})
		}
		req.Attributes[sdkCommon.URLRawQuery] = c.attributes
	}
	req.Type = dr.Properties.ValueType
	reqs = append(reqs, req)

	// execute protocol-specific read operation
	driver := container.ProtocolDriverFrom(c.dic.Get)
	results, err := driver.HandleReadCommands(c.device.Name, c.device.Protocols, reqs)
	if err != nil {
		errMsg := fmt.Sprintf("error reading DeviceResourece %s for %s", dr.Name, c.device.Name)
		return res, errors.NewCommonEdgeX(errors.KindServerError, errMsg, err)
	}

	// convert CommandValue to Event
	res, e = transformer.CommandValuesToEventDTO(results, c.device.Name, dr.Name, c.dic)
	if e != nil {
		return res, errors.NewCommonEdgeX(errors.KindServerError, "failed to convert CommandValue to Event", e)
	}

	return
}

func (c *CommandProcessor) ReadDeviceCommand() (res *dtos.Event, e errors.EdgeX) {
	// 通过给定的profileName和sourceName返回设备命令实例并且检查该设备命令是否存在 DeviceCommand见profiles.go
	dc, ok := cache.Profiles().DeviceCommand(c.device.ProfileName, c.sourceName)
	// 不存在
	if !ok {
		errMsg := fmt.Sprintf("deviceCommand %s not found", c.sourceName)
		return res, errors.NewCommonEdgeX(errors.KindEntityDoesNotExist, errMsg, nil)
	}
	// check deviceCommand is not write-only
	// 检查设备命令是否为只写
	if dc.ReadWrite == common.ReadWrite_W {
		errMsg := fmt.Sprintf("deviceCommand %s is marked as write-only", dc.Name)
		return res, errors.NewCommonEdgeX(errors.KindNotAllowed, errMsg, nil)
	}
	// check ResourceOperation count does not exceed MaxCmdOps defined in configuration
	// 检查资源操作计数是否超过配置中定义的MaxCmdOps,如果超过,返回错误
	configuration := container.ConfigurationFrom(c.dic.Get)
	if len(dc.ResourceOperations) > configuration.Device.MaxCmdOps {
		errMsg := fmt.Sprintf("GET command %s exceed device %s MaxCmdOps (%d)", dc.Name, c.device.Name, configuration.Device.MaxCmdOps)
		return res, errors.NewCommonEdgeX(errors.KindServerError, errMsg, nil)
	}

	lc := bootstrapContainer.LoggingClientFrom(c.dic.Get)
	lc.Debugf("Application - readCmd: reading cmd: %s; %s: %s", dc.Name, common.CorrelationHeader, c.correlationID)

	// prepare CommandRequests
	// 准备命令请求
	reqs := make([]sdkModels.CommandRequest, len(dc.ResourceOperations))
	// 遍历资源操作切片
	for i, op := range dc.ResourceOperations {
		// 设备资源名
		drName := op.DeviceResource
		// check the deviceResource in ResourceOperation actually exist
		// 检查该设备资源在资源操作map中是否真实存在(也就是判断这个设备资源是否可以执行给定操作)，返回设备资源实例
		dr, ok := cache.Profiles().DeviceResource(c.device.ProfileName, drName)
		if !ok {
			errMsg := fmt.Sprintf("deviceResource %s in GET commnd %s for %s not defined", drName, dc.Name, c.device.Name)
			return res, errors.NewCommonEdgeX(errors.KindServerError, errMsg, nil)
		}
		// 保存设备请求
		reqs[i].DeviceResourceName = dr.Name
		reqs[i].Attributes = dr.Attributes
		if c.attributes != "" {
			if len(reqs[i].Attributes) <= 0 {
				reqs[i].Attributes = make(map[string]interface{})
			}
			reqs[i].Attributes[sdkCommon.URLRawQuery] = c.attributes
		}
		reqs[i].Type = dr.Properties.ValueType
	}

	// execute protocol-specific read operation
	// 执行特定协议的读取操作
	// 获取特定协议的驱动实例
	driver := container.ProtocolDriverFrom(c.dic.Get)
	// 处理读命令请求，返回结果
	// HandleReadCommands在 simpledriver.go 中实现
	results, err := driver.HandleReadCommands(c.device.Name, c.device.Protocols, reqs)
	if err != nil {
		errMsg := fmt.Sprintf("error reading DeviceCommand %s for %s", dc.Name, c.device.Name)
		return res, errors.NewCommonEdgeX(errors.KindServerError, errMsg, err)
	}

	// convert CommandValue to Event
	// 将命令数据转为事件
	res, e = transformer.CommandValuesToEventDTO(results, c.device.Name, dc.Name, c.dic)
	if e != nil {
		return res, errors.NewCommonEdgeX(errors.KindServerError, "failed to transform CommandValue to Event", e)
	}

	return
}

// WriteDeviceResource 写设备资源
func (c *CommandProcessor) WriteDeviceResource() (e errors.EdgeX) {
	// 通过给定的profileName和sourceName返回设备资源实例并且检查该设备命令是否存在 DeviceCommand见profiles.go
	dr, ok := cache.Profiles().DeviceResource(c.device.ProfileName, c.sourceName)
	if !ok {
		errMsg := fmt.Sprintf("deviceResource %s not found", c.sourceName)
		return errors.NewCommonEdgeX(errors.KindEntityDoesNotExist, errMsg, nil)
	}
	// check deviceResource is not read-only
	// 检查设备命令是否为只读
	if dr.Properties.ReadWrite == common.ReadWrite_R {
		errMsg := fmt.Sprintf("deviceResource %s is marked as read-only", dr.Name)
		return errors.NewCommonEdgeX(errors.KindNotAllowed, errMsg, nil)
	}

	lc := bootstrapContainer.LoggingClientFrom(c.dic.Get)
	lc.Debugf("Application - writeDeviceResource: writing deviceResource: %s; %s: %s", dr.Name, common.CorrelationHeader, c.correlationID)

	// check request body contains provided deviceResource
	// 检查请求体包含给定的设备资源
	v, ok := c.setParamsMap[dr.Name]
	if !ok {
		if dr.Properties.DefaultValue != "" {
			v = dr.Properties.DefaultValue
		} else {
			errMsg := fmt.Sprintf("deviceResource %s not found in request body and no default value defined", dr.Name)
			return errors.NewCommonEdgeX(errors.KindServerError, errMsg, nil)
		}
	}

	// create CommandValue
	// 创建命令数据结构
	cv, e := createCommandValueFromDeviceResource(dr, v)
	if e != nil {
		return errors.NewCommonEdgeX(errors.KindServerError, "failed to create CommandValue", e)
	}

	// prepare CommandRequest
	// 准备命令请求
	reqs := make([]sdkModels.CommandRequest, 1)
	reqs[0].DeviceResourceName = cv.DeviceResourceName
	reqs[0].Attributes = dr.Attributes
	// 如果命令处理实例的参数不为空，则存入reqs[0].Attributes[sdkCommon.URLRawQuery]
	if c.attributes != "" {
		if len(reqs[0].Attributes) <= 0 {
			reqs[0].Attributes = make(map[string]interface{})
		}
		reqs[0].Attributes[sdkCommon.URLRawQuery] = c.attributes
	}
	reqs[0].Type = cv.Type

	// transform write value
	// 执行写入操作
	configuration := container.ConfigurationFrom(c.dic.Get)
	if configuration.Device.DataTransform {
		e = transformer.TransformWriteParameter(cv, dr.Properties)
		if e != nil {
			return errors.NewCommonEdgeX(errors.KindContractInvalid, "failed to transform set parameter", e)
		}
	}

	// execute protocol-specific write operation
	driver := container.ProtocolDriverFrom(c.dic.Get)
	err := driver.HandleWriteCommands(c.device.Name, c.device.Protocols, reqs, []*sdkModels.CommandValue{cv})
	if err != nil {
		errMsg := fmt.Sprintf("error writing DeviceResourece %s for %s", dr.Name, c.device.Name)
		return errors.NewCommonEdgeX(errors.KindServerError, errMsg, err)
	}

	return nil
}

func (c *CommandProcessor) WriteDeviceCommand() errors.EdgeX {
	dc, ok := cache.Profiles().DeviceCommand(c.device.ProfileName, c.sourceName)
	if !ok {
		errMsg := fmt.Sprintf("deviceCommand %s not found", c.sourceName)
		return errors.NewCommonEdgeX(errors.KindEntityDoesNotExist, errMsg, nil)
	}
	// check deviceCommand is not read-only
	if dc.ReadWrite == common.ReadWrite_R {
		errMsg := fmt.Sprintf("deviceCommand %s is marked as read-only", dc.Name)
		return errors.NewCommonEdgeX(errors.KindNotAllowed, errMsg, nil)
	}
	// check ResourceOperation count does not exceed MaxCmdOps defined in configuration
	configuration := container.ConfigurationFrom(c.dic.Get)
	if len(dc.ResourceOperations) > configuration.Device.MaxCmdOps {
		errMsg := fmt.Sprintf("SET command %s exceed device %s MaxCmdOps (%d)", dc.Name, c.device.Name, configuration.Device.MaxCmdOps)
		return errors.NewCommonEdgeX(errors.KindServerError, errMsg, nil)
	}

	lc := bootstrapContainer.LoggingClientFrom(c.dic.Get)
	lc.Debugf("Application - writeCmd: writing command: %s; %s: %s", dc.Name, common.CorrelationHeader, c.correlationID)

	// create CommandValues
	cvs := make([]*sdkModels.CommandValue, 0, len(c.setParamsMap))
	for _, ro := range dc.ResourceOperations {
		drName := ro.DeviceResource
		// check the deviceResource in ResourceOperation actually exist
		dr, ok := cache.Profiles().DeviceResource(c.device.ProfileName, drName)
		if !ok {
			errMsg := fmt.Sprintf("deviceResource %s in SET commnd %s for %s not defined", drName, dc.Name, c.device.Name)
			return errors.NewCommonEdgeX(errors.KindServerError, errMsg, nil)
		}

		// check request body contains the deviceResource
		value, ok := c.setParamsMap[ro.DeviceResource]
		if !ok {
			if ro.DefaultValue != "" {
				value = ro.DefaultValue
			} else if dr.Properties.DefaultValue != "" {
				value = dr.Properties.DefaultValue
			} else {
				errMsg := fmt.Sprintf("deviceResource %s not found in request body and no default value defined", dr.Name)
				return errors.NewCommonEdgeX(errors.KindServerError, errMsg, nil)
			}
		}

		// ResourceOperation mapping, notice that the order is opposite to get command mapping
		// i.e. the mapping value is actually the key for set command.
		if len(ro.Mappings) > 0 {
			for k, v := range ro.Mappings {
				if v == value {
					value = k
					break
				}
			}
		}

		// create CommandValue
		cv, err := createCommandValueFromDeviceResource(dr, value)
		if err == nil {
			cvs = append(cvs, cv)
		} else {
			return errors.NewCommonEdgeX(errors.KindServerError, "failed to create CommandValue", err)
		}
	}

	// prepare CommandRequests
	reqs := make([]sdkModels.CommandRequest, len(cvs))
	for i, cv := range cvs {
		dr, _ := cache.Profiles().DeviceResource(c.device.ProfileName, cv.DeviceResourceName)

		reqs[i].DeviceResourceName = cv.DeviceResourceName
		reqs[i].Attributes = dr.Attributes
		if c.attributes != "" {
			if len(reqs[i].Attributes) <= 0 {
				reqs[i].Attributes = make(map[string]interface{})
			}
			reqs[i].Attributes[sdkCommon.URLRawQuery] = c.attributes
		}
		reqs[i].Type = cv.Type

		// transform write value
		if configuration.Device.DataTransform {
			err := transformer.TransformWriteParameter(cv, dr.Properties)
			if err != nil {
				return errors.NewCommonEdgeX(errors.KindContractInvalid, "failed to transform set parameter", err)
			}
		}
	}

	// execute protocol-specific write operation
	driver := container.ProtocolDriverFrom(c.dic.Get)
	err := driver.HandleWriteCommands(c.device.Name, c.device.Protocols, reqs, cvs)
	if err != nil {
		errMsg := fmt.Sprintf("error writing DeviceCommand %s for %s", dc.Name, c.device.Name)
		return errors.NewCommonEdgeX(errors.KindServerError, errMsg, err)
	}

	return nil
}

func createCommandValueFromDeviceResource(dr models.DeviceResource, value interface{}) (*sdkModels.CommandValue, errors.EdgeX) {
	var err error
	var result *sdkModels.CommandValue

	v := fmt.Sprint(value)

	// 根据设备资源属性值的类型创建命令参数结构
	switch dr.Properties.ValueType {
	case common.ValueTypeString:
		result, err = sdkModels.NewCommandValue(dr.Name, common.ValueTypeString, v)
	case common.ValueTypeBool:
		var value bool
		// 将value转化为bool类型
		value, err = strconv.ParseBool(v)
		if err != nil {
			errMsg := fmt.Sprintf("failed to convert set parameter %s to ValueType %s", v, dr.Properties.ValueType)
			return result, errors.NewCommonEdgeX(errors.KindServerError, errMsg, err)
		}
		result, err = sdkModels.NewCommandValue(dr.Name, common.ValueTypeBool, value)
	case common.ValueTypeBoolArray:
		var arr []bool
		// 解析json数据存入布尔型arr切片
		err = json.Unmarshal([]byte(v), &arr)
		if err != nil {
			errMsg := fmt.Sprintf("failed to convert set parameter %s to ValueType %s", v, dr.Properties.ValueType)
			return result, errors.NewCommonEdgeX(errors.KindServerError, errMsg, err)
		}
		result, err = sdkModels.NewCommandValue(dr.Name, common.ValueTypeBoolArray, arr)
	case common.ValueTypeUint8:
		var n uint64
		n, err = strconv.ParseUint(v, 10, 8)
		if err != nil {
			errMsg := fmt.Sprintf("failed to convert set parameter %s to ValueType %s", v, dr.Properties.ValueType)
			return result, errors.NewCommonEdgeX(errors.KindServerError, errMsg, err)
		}
		result, err = sdkModels.NewCommandValue(dr.Name, common.ValueTypeUint8, uint8(n))
	case common.ValueTypeUint8Array:
		var arr []uint8
		strArr := strings.Split(strings.Trim(v, "[]"), ",")
		for _, u := range strArr {
			n, err := strconv.ParseUint(strings.Trim(u, " "), 10, 8)
			if err != nil {
				errMsg := fmt.Sprintf("failed to convert set parameter %s to ValueType %s", v, dr.Properties.ValueType)
				return result, errors.NewCommonEdgeX(errors.KindServerError, errMsg, err)
			}
			arr = append(arr, uint8(n))
		}
		result, err = sdkModels.NewCommandValue(dr.Name, common.ValueTypeUint8Array, arr)
	case common.ValueTypeUint16:
		var n uint64
		n, err = strconv.ParseUint(v, 10, 16)
		if err != nil {
			errMsg := fmt.Sprintf("failed to convert set parameter %s to ValueType %s", v, dr.Properties.ValueType)
			return result, errors.NewCommonEdgeX(errors.KindServerError, errMsg, err)
		}
		result, err = sdkModels.NewCommandValue(dr.Name, common.ValueTypeUint16, uint16(n))
	case common.ValueTypeUint16Array:
		var arr []uint16
		strArr := strings.Split(strings.Trim(v, "[]"), ",")
		for _, u := range strArr {
			n, err := strconv.ParseUint(strings.Trim(u, " "), 10, 16)
			if err != nil {
				errMsg := fmt.Sprintf("failed to convert set parameter %s to ValueType %s", v, dr.Properties.ValueType)
				return result, errors.NewCommonEdgeX(errors.KindServerError, errMsg, err)
			}
			arr = append(arr, uint16(n))
		}
		result, err = sdkModels.NewCommandValue(dr.Name, common.ValueTypeUint16Array, arr)
	case common.ValueTypeUint32:
		var n uint64
		n, err = strconv.ParseUint(v, 10, 32)
		if err != nil {
			errMsg := fmt.Sprintf("failed to convert set parameter %s to ValueType %s", v, dr.Properties.ValueType)
			return result, errors.NewCommonEdgeX(errors.KindServerError, errMsg, err)
		}
		result, err = sdkModels.NewCommandValue(dr.Name, common.ValueTypeUint32, uint32(n))
	case common.ValueTypeUint32Array:
		var arr []uint32
		strArr := strings.Split(strings.Trim(v, "[]"), ",")
		for _, u := range strArr {
			n, err := strconv.ParseUint(strings.Trim(u, " "), 10, 32)
			if err != nil {
				errMsg := fmt.Sprintf("failed to convert set parameter %s to ValueType %s", v, dr.Properties.ValueType)
				return result, errors.NewCommonEdgeX(errors.KindServerError, errMsg, err)
			}
			arr = append(arr, uint32(n))
		}
		result, err = sdkModels.NewCommandValue(dr.Name, common.ValueTypeUint32Array, arr)
	case common.ValueTypeUint64:
		var n uint64
		n, err = strconv.ParseUint(v, 10, 64)
		if err != nil {
			errMsg := fmt.Sprintf("failed to convert set parameter %s to ValueType %s", v, dr.Properties.ValueType)
			return result, errors.NewCommonEdgeX(errors.KindServerError, errMsg, err)
		}
		result, err = sdkModels.NewCommandValue(dr.Name, common.ValueTypeUint64, n)
	case common.ValueTypeUint64Array:
		var arr []uint64
		strArr := strings.Split(strings.Trim(v, "[]"), ",")
		for _, u := range strArr {
			n, err := strconv.ParseUint(strings.Trim(u, " "), 10, 64)
			if err != nil {
				errMsg := fmt.Sprintf("failed to convert set parameter %s to ValueType %s", v, dr.Properties.ValueType)
				return result, errors.NewCommonEdgeX(errors.KindServerError, errMsg, err)
			}
			arr = append(arr, n)
		}
		result, err = sdkModels.NewCommandValue(dr.Name, common.ValueTypeUint64Array, arr)
	case common.ValueTypeInt8:
		var n int64
		n, err = strconv.ParseInt(v, 10, 8)
		if err != nil {
			errMsg := fmt.Sprintf("failed to convert set parameter %s to ValueType %s", v, dr.Properties.ValueType)
			return result, errors.NewCommonEdgeX(errors.KindServerError, errMsg, err)
		}
		result, err = sdkModels.NewCommandValue(dr.Name, common.ValueTypeInt8, int8(n))
	case common.ValueTypeInt8Array:
		var arr []int8
		err = json.Unmarshal([]byte(v), &arr)
		if err != nil {
			errMsg := fmt.Sprintf("failed to convert set parameter %s to ValueType %s", v, dr.Properties.ValueType)
			return result, errors.NewCommonEdgeX(errors.KindServerError, errMsg, err)
		}
		result, err = sdkModels.NewCommandValue(dr.Name, common.ValueTypeInt8Array, arr)
	case common.ValueTypeInt16:
		var n int64
		n, err = strconv.ParseInt(v, 10, 16)
		if err != nil {
			errMsg := fmt.Sprintf("failed to convert set parameter %s to ValueType %s", v, dr.Properties.ValueType)
			return result, errors.NewCommonEdgeX(errors.KindServerError, errMsg, err)
		}
		result, err = sdkModels.NewCommandValue(dr.Name, common.ValueTypeInt16, int16(n))
	case common.ValueTypeInt16Array:
		var arr []int16
		err = json.Unmarshal([]byte(v), &arr)
		if err != nil {
			errMsg := fmt.Sprintf("failed to convert set parameter %s to ValueType %s", v, dr.Properties.ValueType)
			return result, errors.NewCommonEdgeX(errors.KindServerError, errMsg, err)
		}
		result, err = sdkModels.NewCommandValue(dr.Name, common.ValueTypeInt16Array, arr)
	case common.ValueTypeInt32:
		var n int64
		n, err = strconv.ParseInt(v, 10, 32)
		if err != nil {
			errMsg := fmt.Sprintf("failed to convert set parameter %s to ValueType %s", v, dr.Properties.ValueType)
			return result, errors.NewCommonEdgeX(errors.KindServerError, errMsg, err)
		}
		result, err = sdkModels.NewCommandValue(dr.Name, common.ValueTypeInt32, int32(n))
	case common.ValueTypeInt32Array:
		var arr []int32
		err = json.Unmarshal([]byte(v), &arr)
		if err != nil {
			errMsg := fmt.Sprintf("failed to convert set parameter %s to ValueType %s", v, dr.Properties.ValueType)
			return result, errors.NewCommonEdgeX(errors.KindServerError, errMsg, err)
		}
		result, err = sdkModels.NewCommandValue(dr.Name, common.ValueTypeInt32Array, arr)
	case common.ValueTypeInt64:
		var n int64
		n, err = strconv.ParseInt(v, 10, 64)
		if err != nil {
			errMsg := fmt.Sprintf("failed to convert set parameter %s to ValueType %s", v, dr.Properties.ValueType)
			return result, errors.NewCommonEdgeX(errors.KindServerError, errMsg, err)
		}
		result, err = sdkModels.NewCommandValue(dr.Name, common.ValueTypeInt64, n)
	case common.ValueTypeInt64Array:
		var arr []int64
		err = json.Unmarshal([]byte(v), &arr)
		if err != nil {
			errMsg := fmt.Sprintf("failed to convert set parameter %s to ValueType %s", v, dr.Properties.ValueType)
			return result, errors.NewCommonEdgeX(errors.KindServerError, errMsg, err)
		}
		result, err = sdkModels.NewCommandValue(dr.Name, common.ValueTypeInt64Array, arr)
	case common.ValueTypeFloat32:
		var val float64
		val, err = strconv.ParseFloat(v, 32)
		if err == nil {
			result, err = sdkModels.NewCommandValue(dr.Name, common.ValueTypeFloat32, float32(val))
			break
		}
		if numError, ok := err.(*strconv.NumError); ok {
			if numError.Err == strconv.ErrRange {
				err = errors.NewCommonEdgeX(errors.KindServerError, "NumError", err)
				break
			}
		}
		var decodedToBytes []byte
		decodedToBytes, err = base64.StdEncoding.DecodeString(v)
		if err == nil {
			var val float32
			val, err = float32FromBytes(decodedToBytes)
			if err != nil {
				break
			} else if math.IsNaN(float64(val)) {
				err = fmt.Errorf("fail to parse %v to float32, unexpected result %v", v, val)
			} else {
				result, err = sdkModels.NewCommandValue(dr.Name, common.ValueTypeFloat32, val)
			}
		}
	case common.ValueTypeFloat32Array:
		var arr []float32
		err = json.Unmarshal([]byte(v), &arr)
		if err != nil {
			errMsg := fmt.Sprintf("failed to convert set parameter %s to ValueType %s", v, dr.Properties.ValueType)
			return result, errors.NewCommonEdgeX(errors.KindServerError, errMsg, err)
		}
		result, err = sdkModels.NewCommandValue(dr.Name, common.ValueTypeFloat32Array, arr)
	case common.ValueTypeFloat64:
		var val float64
		val, err = strconv.ParseFloat(v, 64)
		if err == nil {
			result, err = sdkModels.NewCommandValue(dr.Name, common.ValueTypeFloat64, val)
			break
		}
		if numError, ok := err.(*strconv.NumError); ok {
			if numError.Err == strconv.ErrRange {
				err = errors.NewCommonEdgeX(errors.KindServerError, "NumError", err)
				break
			}
		}
		var decodedToBytes []byte
		decodedToBytes, err = base64.StdEncoding.DecodeString(v)
		if err == nil {
			val, err = float64FromBytes(decodedToBytes)
			if err != nil {
				break
			} else if math.IsNaN(val) {
				err = fmt.Errorf("fail to parse %v to float64, unexpected result %v", v, val)
			} else {
				result, err = sdkModels.NewCommandValue(dr.Name, common.ValueTypeFloat64, val)
			}
		}
	case common.ValueTypeFloat64Array:
		var arr []float64
		err = json.Unmarshal([]byte(v), &arr)
		if err != nil {
			errMsg := fmt.Sprintf("failed to convert set parameter %s to ValueType %s", v, dr.Properties.ValueType)
			return result, errors.NewCommonEdgeX(errors.KindServerError, errMsg, err)
		}
		result, err = sdkModels.NewCommandValue(dr.Name, common.ValueTypeFloat64Array, arr)
	case common.ValueTypeObject:
		result, err = sdkModels.NewCommandValue(dr.Name, common.ValueTypeObject, value)
	default:
		err = errors.NewCommonEdgeX(errors.KindServerError, "unrecognized value type", nil)
	}

	if err != nil {
		return nil, errors.NewCommonEdgeXWrapper(err)
	}
	return result, nil
}

func float32FromBytes(numericValue []byte) (res float32, err error) {
	reader := bytes.NewReader(numericValue)
	err = binary.Read(reader, binary.BigEndian, &res)
	return
}

func float64FromBytes(numericValue []byte) (res float64, err error) {
	reader := bytes.NewReader(numericValue)
	err = binary.Read(reader, binary.BigEndian, &res)
	return
}

```







### device-sdk-go流程梳理

#### 1.生成初始Driver，调用Bootstrap:传入服务名称，设备版本，以及初始Driver

#### 2.进入Bootstrap:生成当前context以及cancel，调用Main，Main传入的参数有:服务名称，服务版本，初始驱动，context,cancel，以及Router

#### 3.进入Main流程：

##### 1）生成定时器（根据服务名称获取系统配置的超时时间以及interval的配置，如果不存在则使用默认值）

##### 2）解析命令行参数

##### 3）设置服务名 服务名为name+os.Getenv("EDGEX_INSTANCE_NAME")

##### 4）生成初始设备服务实例(空)

##### 5）初始化设备服务实例(用户必须提供服务名，其他内容由sdk内部提供)

```go
s.ServiceName = serviceName //由用户指定
sdkCommon.ServiceVersion = serviceVersion  //由makefile指定
s.driver = driver //此处断言成sdk内部提供的ProtocolDriver接口 只需要用户实现接口中的所有方法即可断言成功
s.discovery = discovery //可以没有
s.deviceService = &models.DeviceService{}  //初始化models.DeviceService （注意对应s.deviceService）
s.config = &config.ConfigurationStruct{}   //初始化ConfigurationStruct
```

##### 6）保存命令行参数

```
ds.flags=sdkFlags
```

##### 7）初始化容器（关键就是存入ServiceConstructorMap）

```go
//注意：比如container.ConfigurationName
//var ConfigurationName = di.TypeInstanceToName(config.ConfigurationStruct{})
//其name是PkgPath.interfaceName or PkgPath.non-interfaceName组成
//当第一次调用比如
//ct:=container.TConfigurationFrom(dic.Get) 会通过构造器初始化ConfigurationStruct实例并存储
ds.dic = di.NewContainer(di.ServiceConstructorMap{
		container.ConfigurationName: func(get di.Get) interface{} {
			return ds.config
		},
		container.DeviceServiceName: func(get di.Get) interface{} {
			return ds.deviceService
		},
		container.ProtocolDriverName: func(get di.Get) interface{} {
			return ds.driver
		},
		container.ProtocolDiscoveryName: func(get di.Get) interface{} {
			return ds.discovery
		},
	})
```

##### 8）初始化httpServer

```go
httpServer := handlers.NewHttpServer(router, true)
```

##### 9）调用bootstrap.Run，传入参数如下

```go
bootstrap.Run(
   ctx,
   cancel,
   sdkFlags,
   ds.ServiceName,
   common.ConfigStemDevice,
   ds.config,
   startupTimer,
   ds.dic,
   true,
   []interfaces.BootstrapHandler{
      httpServer.BootstrapHandler,
      messaging.BootstrapHandler,
      clients.BootstrapHandler,
      autoevent.BootstrapHandler,
      NewBootstrap(router).BootstrapHandler,
      autodiscovery.BootstrapHandler,
      handlers.NewStartMessage(serviceName, serviceVersion).BootstrapHandler,
   })


func Run(
	ctx context.Context,
	cancel context.CancelFunc,
	commonFlags flags.Common,
	serviceKey string,
	configStem string,
	serviceConfig interfaces.Configuration,
	startupTimer startup.Timer,
	dic *di.Container,
	useSecretProvider bool,
	handlers []interfaces.BootstrapHandler) {

	wg, deferred, _ := RunAndReturnWaitGroup(
		ctx,
		cancel,
		commonFlags,
		serviceKey,
		configStem,
		serviceConfig,
		nil,
		startupTimer,
		dic,
		useSecretProvider,
		handlers,
	)

	defer deferred()

	// wait for go routines to stop executing.
	wg.Wait()
}

//RunAndReturnWaitGroup引导应用程序。
//它加载配置并调用提供的处理程序列表。任何长时间运行的进程都应该作为go例程在处理程序中生成。
//处理程序应立即返回。调用所有处理程序后，此函数将返回同步。
//对调用方的WaitGroup引用。
//调用者在对返回的引用调用Wait（）之前采取任何有意义的附加操作，以等待应用程序停止（以及在各种处理程序中生成的相应goroutine完全停止）。
func RunAndReturnWaitGroup(
	ctx context.Context,
	cancel context.CancelFunc,
	commonFlags flags.Common,
	serviceKey string,
	configStem string,
	serviceConfig interfaces.Configuration,
	configUpdated config.UpdatedStream,
	startupTimer startup.Timer,
	dic *di.Container,
	useSecretProvider bool,
	handlers []interfaces.BootstrapHandler) (*sync.WaitGroup, Deferred, bool) {

	var err error
	var wg sync.WaitGroup
	deferred := func() {}

	// 检查该设备服务是否提供了初始化的日志客户端，如果没有就创建一个新的，并且将其存入容器
	lc := container.LoggingClientFrom(dic.Get)
    // 这边设备服务启动的时候 容器返回的lc是给空的loggingClient实例，通过服务名和日志等级将其进行初始化，并且通过
    // Update方法这个初始化的Log客户端存入容器中
	if lc == nil {
		lc = logger.NewClient(serviceKey, models.InfoLog)
		dic.Update(di.ServiceConstructorMap{
			container.LoggingClientInterfaceName: func(get di.Get) interface{} {
				return lc
			},
		})
	}
	
    // TranslateInterruptoCancel生成一个go例程，将接收到的系统SIGTERM信号转换为取消引导上下文的cancel的调用。
    // 关键函数 signal.Notify(signalStream, os.Interrupt, syscall.SIGTERM) 监控系统中断信号
	translateInterruptToCancel(ctx, &wg, cancel)
	
    // 读出系统环境变量 并且传入初始化的logClient
	envVars := environment.NewVariables(lc)

    // SecretProvider 被初始化并作为处理配置的一部分放置在容器中，因为需要使用它来获取配置提供程序的访问令牌
    // 并且必须等到从文件加载配置后才能对其进行初始化。
    // commonFlags 即传入的sdkFlags
    // envVars 系统环境变量
    // startupTimer启动计时器
    // context
    // sync.WaitGroup
    // configUpdated config.UpdatedStream 定义接收配置更新时由 ListenForChanges 通知的流类型  此处为nil
    // dic 容器
	configProcessor := config.NewProcessor(commonFlags, envVars, startupTimer, ctx, &wg, configUpdated, dic)
    
    // servicekey serviceName
    // configStem ConfigStemDevice  = "edgex/devices/"
    // serviceConfig  *config.ConfigurationStruct
    
    if err := configProcessor.Process(serviceKey, configStem, serviceConfig, useSecretProvider); err != nil{
		fatalError(err, lc)
	}

	var registryClient registry.Client

	envUseRegistry, wasOverridden := envVars.UseRegistry()
	if envUseRegistry || (commonFlags.UseRegistry() && !wasOverridden) {
		registryClient, err = registration.RegisterWithRegistry(
			ctx,
			startupTimer,
			serviceConfig,
			lc,
			serviceKey,
			dic)
		if err != nil {
			fatalError(err, lc)
		}

		deferred = func() {
			lc.Info("Un-Registering service from the Registry")
			err := registryClient.Unregister()
			if err != nil {
				lc.Error("Unable to Un-Register service from the Registry", "error", err.Error())
			}
		}
	}

	dic.Update(di.ServiceConstructorMap{
		container.ConfigurationInterfaceName: func(get di.Get) interface{} {
			return serviceConfig
		},
		container.RegistryClientInterfaceName: func(get di.Get) interface{} {
			return registryClient
		},
		container.CancelFuncName: func(get di.Get) interface{} {
			return cancel
		},
	})

	// call individual bootstrap handlers.
	startedSuccessfully := true
	for i := range handlers {
		if handlers[i](ctx, &wg, startupTimer, dic) == false {
			cancel()
			startedSuccessfully = false
			break
		}
	}

	return &wg, deferred, startedSuccessfully
}
```



##### Process

```go
func (cp *Processor) Process(
   serviceKey string,
   configStem string,
   serviceConfig interfaces.Configuration, //serviceConfig 实现了interfaces.Configuration的所有方法
   useSecretProvider bool) error {

   // Create some shorthand for frequently used items
   envVars := cp.envVars

   cp.overwriteConfig = cp.flags.OverwriteConfig()

   // 如果需要注册表配置信息或者需要将其推送到配置提供程序，则必须首先加载本地配置。
   if err := cp.loadFromFile(serviceConfig, "service"); err != nil {
      return err
   }

   // 使用 envVars 变量覆盖基于文件的配置。
   // 变量覆盖优先于所有其他变量，因此请确保在使用 config 之前之前应用它们。
   overrideCount, err := envVars.OverrideConfiguration(serviceConfig)
   if err != nil {
      return err
   }

   // 现在配置已经从文件加载并应用了覆盖，可以初始化秘密提供者并将其添加到 DIC，但前提是它被配置为使用。
   var secretProvider interfaces.SecretProvider
   if useSecretProvider {
      secretProvider, err = secret.NewSecretProvider(serviceConfig, cp.ctx, cp.startupTimer, cp.dic)
      if err != nil {
         return fmt.Errorf("failed to create SecretProvider: %s", err.Error())
      }
   }
    
   configProviderUrl := cp.flags.ConfigProviderUrl()

   // Create new ProviderInfo and initialize it from command-line flag or Variables
   configProviderInfo, err := NewProviderInfo(cp.envVars, configProviderUrl)
   if err != nil {
      return err
   }

   switch configProviderInfo.UseProvider() {
   case true:
      var accessToken string
      var getAccessToken types.GetAccessTokenCallback

      // secretProvider will be nil if not configured to be used. In that case, no access token required.
      if secretProvider != nil {
         // Define the callback function to retrieve the Access Token
         getAccessToken = func() (string, error) {
            accessToken, err = secretProvider.GetAccessToken(configProviderInfo.serviceConfig.Type, serviceKey)
            if err != nil {
               return "", fmt.Errorf(
                  "failed to get Configuration Provider (%s) access token: %s",
                  configProviderInfo.serviceConfig.Type,
                  err.Error())
            }

            cp.lc.Infof("Using Configuration Provider access token of length %d", len(accessToken))
            return accessToken, nil
         }

      } else {
         cp.lc.Info("Not configured to use Config Provider access token")
      }

      configClient, err := cp.createProviderClient(serviceKey, configStem, getAccessToken, configProviderInfo.ServiceConfig())
      if err != nil {
         return fmt.Errorf("failed to create Configuration Provider client: %s", err.Error())
      }

      for cp.startupTimer.HasNotElapsed() {
         if err := cp.processWithProvider(
            configClient,
            serviceConfig,
            overrideCount,
         ); err != nil {
            cp.lc.Error(err.Error())
            select {
            case <-cp.ctx.Done():
               return errors.New("aborted Updating to/from Configuration Provider")
            default:
               cp.startupTimer.SleepForInterval()
               continue
            }
         }

         break
      }

      cp.listenForChanges(serviceConfig, configClient)

      cp.dic.Update(di.ServiceConstructorMap{
         container.ConfigClientInterfaceName: func(get di.Get) interface{} {
            return configClient
         },
      })

   case false:
      cp.logConfigInfo("Using local configuration from file", overrideCount)
   }

   // Now that configuration has been loaded and overrides applied the log level can be set as configured.
   err = cp.lc.SetLogLevel(serviceConfig.GetLogLevel())

   return err
}
```