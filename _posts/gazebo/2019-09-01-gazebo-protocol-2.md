---
title: Gazebo服务端与客户端通信协议(二)
categories:
 - gazebo
tags:
 - gazebo
 - protocol
---

# Gazebo 通信协议

上一节我们主要介绍了如何获取GzServer的相关信息以及如何向Server发送消息，这一节主要介绍如何从服务端订阅话题消息。

<!-- more -->

## 订阅话题

之前我们提及在Gazebo中，消息的传递是通过点对点的，所以我们如果要想订阅一个话题，就需要先和Master节点通信，询问我们要订阅的话题的IP地址以及端口。然后与话题的发送端建立连接并获取数据。

### 注册： 客户端->Master节点

发送消息结构如下：

```
| Header (8 Byte) | Packet 对象 |
```

其中Header是用ASCII码表示的十六进制数据，Packet对象是一段protobuf编码，其长度是Header所表示的大小，Packet的格式为

```
message Packet
{
  required Time stamp            = 1;
  required string type           = 2;
  required bytes serialized_data = 3;
}
```

其中`type="subscribe"`，`serialized_data`是`Subscribe`的`protobuf` 编码消息

```
message Subscribe
{
  required string topic    = 1;
  required string host     = 2;
  required uint32 port     = 3;
  required string msg_type = 4;
  optional bool latching   = 5 [default=false];
}
```

其中

- topic : 订阅话题名称
- host : 用来接收Master回复的TCP服务端IP地址
- port : 用来接收Master回复的TCP服务端端口
- msg_type : 话题数据类型
- latching : 当这个标识为true时，在连接时会收到最近一次的数据，如果是false，则仅会收到下一次数据

### 回复：Master节点->客户端

客户端询问后，Master节点会回复相关话题的发送端信息，回复消息结构如下：

```
| Header (8 Byte) | Packet 对象 |
```

其中Header是用ASCII码表示的十六进制数据，Packet对象是一段protobuf编码，其长度是Header所表示的大小，Packet的格式为

```
message Packet
{
  required Time stamp            = 1;
  required string type           = 2;
  required bytes serialized_data = 3;
}
```

其中`type="publisher_subscribe"`，`serialized_data`是`Publish`的`protobuf` 编码消息

```
message Publish
{
  required string topic    = 1;
  required string msg_type = 2;
  required string host     = 3;
  required uint32 port     = 4;
}
```

其中

- topic : 话题名
- msg_type : 话题类型名称
- host : 话题发送端TCP服务IP地址
- port : 话题发送端TCP服务端口信息

### 连接发送端

得到话题发送端的地址，就可以创建连接获取数据了。开启TCP端口，指向发送端，发送接收数据请求，数据格式如下：

```
| Header (8 Byte) | Packet 对象 |
```

其中Packet格式与上述相同

```
message Packet
{
  required Time stamp            = 1;
  required string type           = 2;
  required bytes serialized_data = 3;
}
```

其中`type="sub"`， `serialized_data`是`Subscribe`的`protobuf`编码消息

```
message Subscribe
{
  required string topic    = 1;
  required string host     = 2;
  required uint32 port     = 3;
  required string msg_type = 4;
  optional bool latching   = 5 [default=false];
}
```

### 接收数据

发送完请求就可以等待接收数据了，数据格式如下：

```
| Header (8 Byte) | Message Protobuf 编码 |
```

其中Header是Message Protobuf 编码长度的十六进制数的ASCII编码。Message是话题数据类型对应的Protobuf编码

### 注销：客户端->Master节点

得到话题发送端的地址，就可以创建连接获取数据了。开启TCP端口，指向发送端，发送接收数据请求，数据格式如下：

```
| Header (8 Byte) | Packet 对象 |
```

其中Packet格式与上述相同

```
message Packet
{
  required Time stamp            = 1;
  required string type           = 2;
  required bytes serialized_data = 3;
}
```

其中`type="unsubscribe"`，`serialized_data`是`Subscribe`的`protobuf`编码消息

```
message Subscribe
{
  required string topic    = 1;
  required string host     = 2;
  required uint32 port     = 3;
  required string msg_type = 4;
  optional bool latching   = 5 [default=false];
}
```

在注销后，Master会给发布话题的节点发送unsubscribe消息

