---
title: EdgeX的使用
date: 2021-09-03
categories:
- EdgeX
tags:
- EdgeX构建
language: zh-CN
toc: true
---

### EdgeX的使用

演示功能需要包含设备数据采集，数据分析，数据分析后的设备控制命令发送，以及云端的数据导出和远程数据访问显示。  设备目前可以使用edgeX 的虚拟设备。

以虚拟设备为例：edgex自带的虚拟设备应该不会自己时刻产生数据，只有当edgex请求或者用户通过api请求其中的方法时，才会返回数据。在[devices.toml](https://github.com/edgexfoundry/device-virtual-go/blob/main/cmd/res/devices/devices.toml)文件中，对每个设备配置了**DeviceList/DeviceList.AutoEvents**，也就是自动事件，以此完成每个interval从虚拟设备收集数据发送到核心数据。

<!--more-->

![](https://cxd-note-img.oss-cn-hangzhou.aliyuncs.com/typora-note-img/image-20210907231555149.png)

下面的步骤通过postman操作：

##### 数据采集

```
192.168.0.107:59882/api/v2/device/name/Random-Integer-Device/Int8
```

<img src="https://cxd-note-img.oss-cn-hangzhou.aliyuncs.com/typora-note-img/image-20210907144111437.png" style="zoom:67%;" />

##### 数据分析以及数据分析后设备控制命令发送

通过eKuiper 分析来自 EdgeX消息总线的数据

###### 第一步：创建流

```post  http://192.168.0.107:59720/streams```

```body {"sql": "create stream demo() WITH (FORMAT=\"JSON\", TYPE=\"edgex\")"}```

<img src="https://cxd-note-img.oss-cn-hangzhou.aliyuncs.com/typora-note-img/image-20210907164410143.png"  style="zoom:67%;" />

###### 创建规则

第一条规则：

监视来自`Random-UnsignedInteger-Device`设备的事件，如果`uint8`发现读数值大于`20`事件中的值，则向`Random-Boolean-Device`设备发送命令以开始生成随机数（将随机生成 bool 设置为 true）。

```
#post
http://192.168.0.107:59720/rules
```

```yaml
#body
{
  "id": "rule1",
  "sql": "SELECT uint8 FROM demo WHERE uint8 > 20",
  "actions": [
    {
      "rest": {
        "url": "http://edgex-core-command:59882/api/v2/device/name/Random-Boolean-Device/WriteBoolValue",
        "method": "put",
        "dataTemplate": "{\"Bool\":\"true\", \"EnableRandomization_Bool\": \"true\"}",
        "sendSingle": true
      }
    },
    {
      "log":{}
    }
  ]
}
```

<img src="https://cxd-note-img.oss-cn-hangzhou.aliyuncs.com/typora-note-img/image-20210907164745373.png" style="zoom:67%;" />

第二条规则：

监视来自`Random-Integer-Device`设备的事件，如果`int8`读取值的平均值（20 秒内）大于 0，则向`Random-Boolean-Device`设备发送命令以停止生成随机数字（将随机生成 bool 设置为 false）。

```
#post
http://192.168.0.107:59720/rules
```

```yaml
{
  "id": "rule2",
  "sql": "SELECT avg(int8) AS avg_int8 FROM demo WHERE int8 != nil GROUP BY TUMBLINGWINDOW(ss, 20) HAVING avg(int8) > 0",
  "actions": [
    {
      "rest": {
        "url": "http://edgex-core-command:59882/api/v2/device/name/Random-Boolean-Device/WriteBoolValue",
        "method": "put",
        "dataTemplate": "{\"Bool\":\"false\", \"EnableRandomization_Bool\": \"false\"}",
        "sendSingle": true
      }
    },
    {
      "log":{}
    }
  ]
}
```

<img src="https://cxd-note-img.oss-cn-hangzhou.aliyuncs.com/typora-note-img/image-20210907170228371.png"  style="zoom:67%;" />

###### 打印eKuiper日志

```docker logs edgex-kuiper```

![](https://cxd-note-img.oss-cn-hangzhou.aliyuncs.com/typora-note-img/image-20210907170649143.png)

测试完成

换个例子

删除刚刚创建的两条规则，只创建一条规则，作用是监视来自`Random-Integer-Device`设备的事件，如果`int8`读取值的平均值（20 秒内）小于30，则向`Random-Boolean-Device`设备发送命令以开始生成随机数字，如果`int8`读取值的平均值（20 秒内）大于等于30，则向`Random-Boolean-Device`设备发送命令以停止生成随机数字。

```
#post
http://192.168.0.107:59720/rules
```

```yaml
#body
{
  "id": "rule2",
  "sql": "SELECT avg(int8) AS avg_int8 FROM demo WHERE int8 != nil GROUP BY TUMBLINGWINDOW(ss, 20)",
  "actions": [
    {
      "rest": {
        "url": "http://edgex-core-command:59882/api/v2/device/name/Random-Boolean-Device/WriteBoolValue",
        "method": "put",   
        "dataTemplate": "{\"Bool\":\"{{if lt .avg_int8 30.0}}true\"{{else if ge .avg_int8 30.0}}false\"{{end}},\"EnableRandomization_Bool\": \"{{if lt .avg_int8 30.0}}true\"{{else if ge .avg_int8 30.0}}false\"{{end}}}",
        "sendSingle": true
      }
    },
    {
      "log":{}
    }
  ]
}
```

<img src="https://cxd-note-img.oss-cn-hangzhou.aliyuncs.com/typora-note-img/image-20210907215355960.png" style="zoom:67%;" />

这里面需要用到go template模板，看到Stack Overflow中有提到现在依然不是非常完善，因此在使用的时候比较麻烦。比如这边的30必须写出30.0，不然就会报错。

测试成功

<img src="https://cxd-note-img.oss-cn-hangzhou.aliyuncs.com/typora-note-img/image-20210907225609140.png"  style="zoom:67%;" />

##### 云端的数据导出和远程数据访问显示

首先将将应用程序服务添加到 docker-compose.yml文件中，设置如下：

<img src="https://cxd-note-img.oss-cn-hangzhou.aliyuncs.com/typora-note-img/image-20210908122518281.png" style="zoom:67%;" />

运行```docker-compose up -d ```更新服务，之后在浏览器中打开http://www.hivemq.com/demos/websocket-client

connect之后订阅主题Cxdtest，即可得到发送至EdgeX核心数据的相关数据。

<img src="https://cxd-note-img.oss-cn-hangzhou.aliyuncs.com/typora-note-img/image-20210908122835643.png"  style="zoom:67%;" />