
### 介绍

此文档涵盖了Kafka 0.8及之前版本的通讯协议实现。其目的是提供一个包含的可请求的协议及其二进制格式以及如何正确使用他们来实现一个客户端的通讯协议文档。本文假设您已经了解了Kafka基本的设计以及术语。

0.7和更早的版本所使用的协议与此类似，但我们（希望）通过一次性地斩断兼容性，以便清理原有设计上的沉疴，并且泛化一些概念。

### 概述

Kafka 协议相对比较简单，仅仅只有六类核心交互 API

- Metadata - 描述当前存在的 broker, 包括它们的 host 和端口的信息，并且包含分区和节点的映射关系
- Send - 发送消息到 broker
- Fetch - 从 broker 中获取消息，分别是获取数据、获取集群 meta 信息、获取 offset 信息
- Offsets - 获取给定 topic 分区对应的存在的 offset 信息
- Offset Commit - 提交一个消费分组对应的 offset
- Offset Fetch - 获取一个消费分组对应的 offset

每个类型将在下面详细介绍。并且在 0.9 以后 kafka 增加消费组管理功能，客户端 API 由5个请求组成：

- GroupCoordinator - 分配一个组的当前协调器
- JoinGroup - 加入一个组，如果其不存在这创建
- SyncGroup - 同步组内成员的所有状态值 (例 消费者分配分区)
- Heartbeat - 保持一个消费成员在改组内的心跳
- LeaveGroup - 直接离开一个组

最后，提供了管理端的 API 接口，用来监控、管理 kafka 集群

- DescribeGroups - 用来检查当前组成员的状态 (例 当前消费者分区分配情况)
- ListGroups - 列出通过 broker 管理的消费组信息

### 起步

#### 网络

Kafka 使用的是 TCP 上的二进制协议，协议定义了所有的 API 的请求响应结构。所有的消息没有大小限制并且通过下面介绍的原始类型组成

客户端初始化连接，然后写入请求的消息序列并且读取关联的响应消息序列，连接和断开时都不需要握手操作，如果需要保持一个长连接，那么 TCP 协议本身将会节省很多TCP握手时间，但如果真的重新建立连接，那么代价也相当小。

客户端需要和多个 broker 之间保持连接，因为数据是被分区的，客户端需要和对应数据所在的 broker 交互。一般而言，单个 broker 和单个客户端没有比较保持多个连接 （例如 连接池）

服务端保证单个 TCP 连接，所有的请求发送顺序和最终响应的顺序的一致的，为了保证处理顺序一致，单个连接请求同时只会处理一个请求指令。客户端可以使用非阻塞方式来提升吞吐量。也就说客户端在等待请求响应的时候可以同时发送下一个处理请求，因为待完成的请求将会在底层操作系统套接字缓冲区进行缓冲。除非特别说明，所有的请求是由客户端启动，并从服务器获取到相应的响应消息。

服务器可以配置请求大小的限制，如果超过大小限制连接将断开

#### 分区与引导

Kafka 是一个分区系统，不是所有的服务器中拥有全部的数据集合。topic 会被分为 P（创建分区预定义好的） 个分A区, 每个分区的复制因子是 N， Topic Partition根据顺序在“提交日志”中编号为0，1，…，P。

所有具备该特性的系统，都会有一个如何将指定的数据分配到指定分区中的问题。Kafka 客户端直接控制分配，broker 自己没有特别的语义来决定分配到那个分区。相反，生产者直接将消息发送到指定的分区并且在消费消息也要从制定分区中读取。如果两个客户端使用同一个分区描述，那么他们必须使用同一方式计算发送消息和对应分区的映射

生产消息或者消费消息请求必须发送到当前给定分区 leader 角色的 broker 节点上。这个是 broker 强制的条件，如果一个请求发送到一个分区错误的 broker 节点是将会响应 `NotLeaderForPartition` 错误码(下面会描述)

如何知道现有存在的 topic， 它们存在的分区数，以及当前分区对应的 broker, 可以直接将请求发送到正确的 broker 节点？这些信息都是动态变化的，所以不能简单的配置到一个静态映射配置文件中。所有的 kafka broker 节点提供获取 meta 数据请求接口，返回描述当前集群的状态信息: 存在的所有 topic、每个 topic 分区信息、每个分区对应的 leader 节点以及 broker 节点的ip和端口信息

换句话就是，客户端需要通过需要通过其中一个 broker 来获取所有存在的 broker 信息以及分区信息。第一个 broker 可能自己会挂掉，所以最好的实践方案是采用一个 broker 列表来启动。在客户端中用户可以选择使用负载均衡或者静态的配置两个或三个 host 

客户端不需要保持轮询的方式查看集群是否变更，可以获取 meta 信息并且做好缓存，知道接收到错误或者 meta 缓存信息过期是去再次获取同步信息。这个错误一般在下面几种形式下会发生：

- 连接错误，客户端无法与指定的 broker 交互
- 请求响应错误，该broker 不在是请求中指定分区的 leader broker

通过循环 bootstrap 地址，知道找到一个可以连接的并且获取集群的 meta 信息, 处理消费或者生产的请求，并且通过 topic 和分区信息确定合适的交互 broker 进行生产和消费, 如果获取到一个错误，重新刷新 meta 信息

#### 分区策略

上面提到分配消息到分区在生产客户端控制，那么该如何设计并且暴露给最终用户？

分区实际上在 Kafka 中有两个目的：

- 均衡 broker 节点的数据和请求负载
- 它允许多个消费者之间处理分发消息的同时，能够维护本地状态，并且在分区中维持消息的顺序。我们称这种语义的分区（semantic partitioning）。

对于给定的使用场景下，你可能只关心其中的一个或两个。

为了实现简单的负载均衡，对于客户端来说一个简单的策略就是轮询请求所有的 broker. 另外一个选择，在生产者大于 broker 节点的场景中，给每个客户机随机选择并发布消息到该分区, 后者的策略可以减少 TCP 连接

语义分区是指使用关键字来决定消息分配的分区。例如，如果你正在处理一个点击消息流时，可能需要用户 ID 分流，使得特定用户的所有数据被单个消费者消费。要做到这一点，客户端可以采取与消息关联的关键字，并使用关键字的某个 hash 值来选择传送的分区


#### 批量

Our apis encourage batching small things together for efficiency. We have found this is a very significant performance win. Both our API to send messages and our API to fetch messages always work with a sequence of messages not a single message to encourage this. A clever client can make use of this and support an "asynchronous" mode in which it batches together messages sent individually and sends them in larger clumps. We go even further with this and allow the batching across multiple topics and partitions, so a produce request may contain data to append to many partitions and a fetch request may pull data from many partitions all at once.
The client implementer can choose to ignore this and send everything one at a time if they like.

#### 版本与兼容性
 
The protocol is designed to enable incremental evolution in a backward compatible fashion. Our versioning is on a per-api basis, each version consisting of a request and response pair. Each request contains an API key that identifies the API being invoked and a version number that indicates the format of the request and the expected format of the response.
The intention is that clients would implement a particular version of the protocol, and indicate this version in their requests. Our goal is primarily to allow API evolution in an environment where downtime is not allowed and clients and servers cannot all be changed at once.
The server will reject requests with a version it does not support, and will always respond to the client with exactly the protocol format it expects based on the version it included in its request. The intended upgrade path is that new features would first be rolled out on the server (with the older clients not making use of them) and then as newer clients are deployed these new features would gradually be taken advantage of.
Currently all versions are baselined at 0, as we evolve these APIs we will indicate the format for each version individually.

### 协议

#### 协议原始类型

#### 请求语法要点

#### 通用请求响应结构

##### 请求

##### 响应

##### 消息集合

##### 压缩

#### API
