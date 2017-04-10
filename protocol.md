
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

我们鼓励小批量的调用 API 接口，这样对性能提升比较大。在发送或者消费消息的时候建议批量操作一个消息序列而不建议单条操作。一个优秀的客户端是需要支持异步模式方式独立的发送或者批量发送消息。进一步的允许批量跨多个 topic 或者多个分区，对于生产消息请求可能会发送到多个分区或者在消费请求的时候会从多个分区获取一次性。

客户端实现者可以选择性的忽略这个，所有的消息一次只发送一个

#### 版本与兼容性
 
该协议设计的目的是为了达到在向后兼容的基础上渐进演化，我们的版本是基于每个 API 基础之上的，在请求和响应对都包含一个版本信息。每个请求中包含 API key 里面包含了被调用的标识，以及标识这些请求和响应格式的版本号

这样做的目的是为了在客户端实现指定版本的协议是，可以在请求中明确表示版本。目标主要是为了在不允许停机的环境下进行更新，这种环境下，客户端和服务器不能一次性都切换所使用的API。

如果服务端不支持某个版本是将拒绝请求，并始终返回它期望收到的能够完成请求响应的版本的协议格式.预期的升级路径方式是，新功能将首先部署到服务器（老客户端无法完全利用他们的新功能），然后随着新的客户端的部署，这些新功能将逐步被利用。

目前，所有版本基线为0，当我们演进这些API时，我们将分别显示每个版本的格式。

### 协议

#### 协议原始类型

该协议建立在如下几种基础类型上

定长基本类型

int8, int16, int32, int64 - 不同精度的有符号整形以大端方式存储

变长类型

bytes, string - 该类型是有一个有符号整型 N 和后面跟着 N个字节的数据组成。如果长度是 -1 表示 `null`. 字符串使用 int16 表示长度，bytes 使用 int32 表示

数组

该类型是用来表示重复的数据结构。他们总是使用一个代表元素个数int32整数N，以及后续的N个重复结构体组成，这些数据本身又由其他的基础类型组成。后面会用 BNF 语法展示一个数组结构 `foo` 为 [foo]

#### 请求语法要点

后面的 BNF 会确切的以上下文无关的语法描述了请求和响应的二进制格式。每个 API 都会一起给出请求和响应，以及所有的子定义。BNF 使用没有压缩便于阅读的名称（例如使用符号化的名称定义生产者的错误码，尽管仅仅需要 int16 表示），一般在一个 BNF,一个序列标识一个关联关系，例如在一个 MetadataRequest 中会给出一序列字节的组成，首先是一个 VersionId, 然后是 ClientId, 和一个有数组组成的 TopicNames (每个都其自身的定义)。自定义一般会使用驼峰的方式，而基础类型使用小写方式标识，当可能存在多用自定义类型是使用 `|` 分隔，并且使用括号病史分组。顶级定义不缩进，后续字部分会缩进

#### 通用请求响应结构

所有请求和响应都会遵守如下语法，其余的会在下文增量介绍

```
RequestOrResponse => Size (RequestMessage | ResponseMessage)
	Size => int32
```

|字段 | 描述 |
|-----|-----|
| MessageSize |  MessageSize 这个字段给定一个请求或者响应消息数据的长度。客户端可以首先读取 4个字节的整型来获取长度，然后在读取解析后面 N 个字节的请求内容 |

##### 请求

如下是所有的请求格式：

```
RequestMessage => ApiKey ApiVersion CorrelationId ClientId RequestMessage
	ApiKey => int16
	ApiVersion => int16
	CorrelationId => int32
	ClientId => string
	RequestMessage => MetadataRequest | ProduceRequest | FetchRequest | OffsetRequest | OffsetCommitRequest | OffsetFetchRequest
```
|字段 | 描述 |
|-----|-----|
| ApiKey |  这是一个数字 ID 调用对应的 API (例如，可能是一个获取 metadata 的请求或者是一个生产消息请求或者是一个消费消息的请求)|
| ApiVersion | 数字版本号对于该 API的，我们为每个 API定义一个版本号，该版本号运行服务端根据版本号正确的解析请求内容。响应格式将会和请求版本一一对应 |
| CorrelationId | 用户提供的一个整形。服务端将在响应的时候返回，不会修改。其作用是用来匹配客户端和服务端之间的请求的 |
| ClientId | 这个是用户提供客户端应用程序的标识的。用户可以使用任何喜欢的标识符，将被应用在错误日志记录、监控统计方面等等。例如你可能不仅仅想统计每秒总的请求数，而且要统计每个客户端的请求数，那它就有用了，这个 ID 起到一个逻辑上分组的作用 | 

下面我们来描述各种请求响应消息

##### 响应

```
Response => CorrelationId ResponseMessage
	CorrelationId => int32
	ResponseMessage => MetadataResponse | ProduceResponse | FetchResponse | OffsetResponse | OffsetCommitResponse | OffsetFetchResponse
```

|字段 | 描述 |
|-----|-----|
| CorrelationId | 服务端返回的关联 ID, 该值是客户端请求时设置的, 不会修改，原样返回的 |

响应将永远和请求时匹配的（例如一个 MetadataResponse 将对应的是对 MetadataRequest 的响应）

##### 消息集合

One structure common to both the produce and fetch requests is the message set format. A message in kafka is a key-value pair with a small amount of associated metadata. A message set is just a sequence of messages with offset and size information. This format happens to be used both for the on-disk storage on the broker and the on-the-wire format.

A message set is also the unit of compression in Kafka, and we allow messages to recursively contain compressed message sets to allow batch compression.

N.B., MessageSets are not preceded by an int32 like other array elements in the protocol.

```
MessageSet => [Offset MessageSize Message]
	Offset => int64
	MessageSize => int32
```

消息格式：

```
v0
Message => Crc MagicByte Attributes Key Value
  Crc => int32
  MagicByte => int8
  Attributes => int8
  Key => bytes
  Value => bytes
 
v1 (supported since 0.10.0)
Message => Crc MagicByte Attributes Key Value
  Crc => int32
  MagicByte => int8
  Attributes => int8
  Timestamp => int64
  Key => bytes
  Value => bytes
```

|字段 | 描述 |
|-----|-----|
| Offset | This is the offset used in kafka as the log sequence number. When the producer is sending non compressed messages, it can set the offsets to anything. When the producer is sending compressed messages, to avoid server side recompression, each compressed message should have offset starting from 0 and increasing by one for each inner message in the compressed message. (see more details about compressed messages in Kafka below) |
| Crc | The CRC is the CRC32 of the remainder of the message bytes. This is used to check the integrity of the message on the broker and consumer. |
| MagicByte | This is a version id used to allow backwards compatible evolution of the message binary format. The current value is 1. |
| Attributes | This byte holds metadata attributes about the message. The lowest 3 bits contain the compression codec used for the message. The fourth lowest bit represents the timestamp type. 0 stands for CreateTime and 1 stands for LogAppendTime. The producer should always set this bit to 0. (since 0.10.0) All other bits should be set to 0. |
| Timestamp | This is the timestamp of the message. The timestamp type is indicated in the attributes. Unit is milliseconds since beginning of the epoch (midnight Jan 1, 1970 (UTC)). |
| Key | The key is an optional message key that was used for partition assignment. The key can be null. |
| Value | The value is the actual message contents as an opaque byte array. Kafka supports recursive messages in which case this may itself contain a message set. The message can be null.|

##### 压缩

Kafka supports compressing messages for additional efficiency, however this is more complex than just compressing a raw message. Because individual messages may not have sufficient redundancy to enable good compression ratios, compressed messages must be sent in special batches (although you may use a batch of one if you truly wish to compress a message on its own). The messages to be sent are wrapped (uncompressed) in a MessageSet structure, which is then compressed and stored in the Value field of a single "Message" with the appropriate compression codec set. The receiving system parses the actual MessageSet from the decompressed value. The outer MessageSet should contain only one compressed "Message" (see KAFKA-1718 for details).
Kafka currently supports two compression codecs with the following codec numbers:

|压缩类型 | 编码 |
|-----|-----|
| None | 0 | 
| GZIP | 1 | 
| Snappy | 2 | 

#### API

This section gives details on each of the individual APIs, their usage, their binary format, and the meaning of their fields.

##### Metadata API

This API answers the following questions:
What topics exist?
How many partitions does each topic have?
Which broker is currently the leader for each partition?
What is the host and port for each of these brokers?
This is the only request that can be addressed to any broker in the cluster.
Since there may be many topics the client can give an optional list of topic names in order to only return metadata for a subset of topics.
The metadata returned is at the partition level, but grouped together by topic for convenience and to avoid redundancy. For each partition the metadata contains the information for the leader as well as for all the replicas and the list of replicas that are currently in-sync.
Note: If "auto.create.topics.enable" is set in the broker configuration, a topic metadata request will create the topic with the default replication factor and number of partitions. 

