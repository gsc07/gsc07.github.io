---
title: Gazebo服务端与客户端通信协议(一)
categories:
 - gazebo
tags:
 - gazebo
 - protocol
---

# Gazebo 通信协议

Gazebo是一个基于物理引擎的可独立运行的模拟器，其主要分为GzServer和GzClient两部分。其中GzServer用于后台物理引擎，碰撞检测，光照渲染等计算；GzClient提供可视化以及用户交互界面。这两者中间通过TCP通信，以及Protobuf （proto2）编码。由于官方并没有给出相应的通讯协议文档，以下通信协议是由根据源码解析获得。

<!-- more -->

## 获取Master相关信息

GzServer启动时会开启一个TCP服务

```
>> gzserver --verbose
Gazebo multi-robot simulator, version 9.10.0
Copyright (C) 2012 Open Source Robotics Foundation.
Released under the Apache 2 License.
http://gazebosim.org

[Msg] Waiting for master.
[Msg] Connected to gazebo master @ http://127.0.0.1:11345
[Msg] Publicized address: 172.16.20.205
```

通过日志我们可以看到Master默认开启一个端口为11345的TCP服务，这个服务用来给客服端提供相应的注册信息，比如话题的开启，关闭，相应话题通信使用的端口等。

我们可以使用socket连接

```
IP:127.0.0.1
Port: 11345
```

当连接成功时，可以收到一段信息，这段信息可以分为**版本信息，话题前缀信息，已注册话题列表** 三部分，三部分数据结构相同，格式定义如下：

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



### 版本信息：

```
30 30 30 30 30 30 32 62 0A 0C 08 AD C5 A2 EB 05 10 C1 D3 F7 E3 01 12 0C 76 65 72 73 69 6F 6E 5F 69 6E 69 74 1A 0D 0A 0B 67 61 7A 65 62 6F 20 39 2E 31 30
```

其中Header部分为

```
30 30 30 30 30 30 32 62 -> ascii [0000002b] -> 十进制数(43)
```

Packet 部分为，其长度为43 Byte

```
0A 0C 08 AD C5 A2 EB 05 10 C1 D3 F7 E3 01 12 0C 76 65 72 73 69 6F 6E 5F 69 6E 69 74 1A 0D 0A 0B 67 61 7A 65 62 6F 20 39 2E 31 30
```

解析后数据为

```
stamp {
  sec: 1567138477
  nsec: 478013889
}
type: "version_init"
serialized_data: "\n\vgazebo 9.10"
```

其中serialized_data是一段Protobuf编码，message类型是

```
message GzString
{
  required string data = 1;
}
```

serialized_data解析后结果为

```
gazebo 9.10
```



### 话题前缀信息

数据解析后结果为

```
stamp {
  sec: 1567138477
  nsec: 478050587
}
type: "topic_namepaces_init"
serialized_data: "\n\adefault\n\a/gazebo"
```

其中serialized_data是一段Protobuf编码，message类型是

```
message GzString_V
{
  repeated string data = 1;
}
```

serialized_data解析后结果为

```
[default, /gazebo]
```



### 已注册话题列表

```
stamp {
  sec: 1567138477
  nsec: 478079156
}
type: "publishers_init"
serialized_data: "\nN\n\037/gazebo/default/pose/local/info\022\030gazebo.msgs.PosesStamped\032\r172.16.20.205 \333\302\002\nH\n\031/gazebo/default/pose/info\022\030gazebo.msgs.PosesStamped\032\r172.16.20.205 \333\302\002\n9\n\023/gazebo/default/gui\022\017gazebo.msgs.GUI\032\r172.16.20.205 \333\302\002\nC\n\030/gazebo/default/response\022\024gazebo.msgs.Response\032\r172.16.20.205 \333\302\002\nM\n\033/gazebo/default/world_stats\022\033gazebo.msgs.WorldStatistics\032\r172.16.20.205 \333\302\002\nB\n\032/gazebo/default/model/info\022\021gazebo.msgs.Model\032\r172.16.20.205 \333\302\002"
```

其中serialized_data是一段Protobuf编码，message类型是

```
message Publishers
{
  repeated Publish publisher = 1;
}

message Publish
{
  required string topic    = 1;
  required string msg_type = 2;
  required string host     = 3;
  required uint32 port     = 4;
}
```

serialized_data解析后结果为

```
topic: "/gazebo/default/pose/local/info"
msg_type: "gazebo.msgs.PosesStamped"
host: "192.168.18.152"
port: 45157

topic: "/gazebo/default/pose/info"
msg_type: "gazebo.msgs.PosesStamped"
host: "192.168.18.152"
port: 45157

topic: "/gazebo/default/gui"
msg_type: "gazebo.msgs.GUI"
host: "192.168.18.152"
port: 45157

topic: "/gazebo/default/response"
msg_type: "gazebo.msgs.Response"
host: "192.168.18.152"
port: 45157

topic: "/gazebo/default/world_stats"
msg_type: "gazebo.msgs.WorldStatistics"
host: "192.168.18.152"
port: 45157

topic: "/gazebo/default/model/info"
msg_type: "gazebo.msgs.Model"
host: "192.168.18.152"
port: 45157
```

## 发布话题

在Gazebo中发布话题是点对点的，所以一般在发送话题前开一个TCP服务用来监听其他节点的订阅，并将这个TCP服务信息在master节点中订阅。当有订阅者时，其会向TCP服务发送消息，此时TCP服务就可以向订阅者发送数据了。

### 注册： 客户端->Master节点

发布话题前需要告知master新增话题发布器，也就是向master服务发送一段内容，内容格式和上面定义格式类似

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

其中`type="advertise"`， `serialized_data`是`Publish`的`protobuf`编码消息

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
- msg_type : 话题类型名称(`protobuf`消息类型名，类似`gazebo.msgs.Vector3d`)
- host : 客户端节点TCP服务IP地址
- port : 客户端节点TCP服务端口信息

### 回复：Master节点->客户端

当master收到客户端节点创建话题发布器的指令后，会给客户端回复相关信息，回复数据格式如下：

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

其中`type="publisher_add"`， `serialized_data`是`Publish`的`protobuf`编码消息，这个数据和你给服务端发送的消息一致

```
topic: "/gazebo/default/my_velodyne/vel_cmd"
msg_type: "gazebo.msgs.Vector3d"
host: "127.0.0.1"
port: 40711
```

### 订阅客户端信息获取

当在master中注册后，其他节点订阅话题的时候，会给当前客户端的TCP服务发送消息，消息数据格式为：

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

当拿到这个socket的时候，就可以直接和订阅者直接通信了

### 发送消息

拿到订阅者的socket之后，就可以直接通过TCP给订阅者发送数据，数据格式为：

```
| Header (8 Byte) | Message Protobuf 编码 |
```

其中Header是Message Protobuf 编码长度的十六进制数的ASCII编码，与上面介绍类似

### 注销 ：客户端->Master节点

当不想再发送话题的时候，我们可以取消发送话题，这个时候需要给Master发送注销指令，格式如下：

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

其中`type="unadvertise"`，`serialized_data`是`Publish`的`protobuf`编码消息

```
message Publish
{
  required string topic    = 1;
  required string msg_type = 2;
  required string host     = 3;
  required uint32 port     = 4;
}
```

### 回复：Master节点->客户端

当Master节点收到注销请求时，会回复确认消息，格式如下：

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

其中`type="publisher_del"`，`type=""`，`serialized_data`是`Publish`的`protobuf`编码消息

```
message Publish
{
  required string topic    = 1;
  required string msg_type = 2;
  required string host     = 3;
  required uint32 port     = 4;
}
```