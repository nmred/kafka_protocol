
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

消息集合格式是一个生产、消费请求的通用格式。在 Kafka 中一个消息由一个 key-value 键值对和一些关联的 meta 信息组成。一个消息集合是有多个消息体、消息的 offset 值以及消息大小组成的数组。该格式将被用在 broker 节点的磁盘存储和传输过程中。

在 kafka 中 一个消息集合也是一个压缩单元, 允许递归的包含已经压缩的消息体。

注意：消息集合不能像协议中其他数组元素先于 int32 之前

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
| Offset | offset 是 kafka 用来标识的序列号, 当生产者发送没有压缩的消息是，可以将设置为任何值。当生产者发送已经压缩的消息，为了避免服务端再次压缩，每个压缩的消息应该设置一个从 0 开始自增长的 offset。（详细参考下面介绍在kafka 压缩消息）|
| Crc |CRC 是循环冗余码校验。用来校验在 broker 和消费者端的消息的完整性 |
| MagicByte | 这个是版本 ID， 用来向后兼容演进消息二进制格式，当前值为 1 |
| Attributes | 该 byte 位会存放一些消息的描述信息。低三位存放消息压缩类型 codec 值。第四位代表时间戳类型，0 代表消息创建时间，1 代表 消息 LogAppendTime 。对于生产者设置为 0 即可。（从 0.10.0 开始）所有的其他为都置为0 |
| Timestamp | 消息的时间戳。依照属性中的时间类型。单位是毫秒从epoch 开始 （UTC）|
| Key | 该值是可选项，其用来作为分区分配使用，可以为 null | 
| Value | 实际的消息内容，由 byte 数组组成。Kafka 支持消息递归，即可能自身包含一个消息集合。消息可以为 null |

##### 压缩

Kafka 支持压缩会有一些优化效果，但是也会比原始消息处理起来负责。因为单个消息可能不会有大的压缩比，压缩的消息必须在特定批量发送（尽管你可能一个批次发送一个消息如果认为压缩有效）。消息在发送的时候会被包装到 MessageSet 结构中，压缩后的消息将存储到单个 Message 结构中的 Value 字段中，并且设置 codec 字段。当接受到消息将被系统的解压缩 value 值。

Kafka 当前支持两种压缩方式，codec 对应值如下：

|压缩类型 | 编码 |
|-----|-----|
| None | 0 | 
| GZIP | 1 | 
| Snappy | 2 | 

#### API

该节将详细的介绍每个 API 的用法、二进制格式以及字段的意义

##### Metadata API

This API answers the following questions:
该 API 将解答如下问题：

- 当前集群存在那些 topic?
- 每个 topic 有多少分区？
- 每个分区当前对应的 leader 节点？
- 每个 broker 对应的 IP 和端口？

该接口可以请求集群中的任意一个节点。

由于可能存在很多 topic, 所以可以选择仅仅返回一个 topic 名称列表以便返回 topic meta 信息

Meta 信息返回时分区级别的，以 topic 为组的方式返回防止冗余。meta 信息中每个分区信息包含 leader 信息以及当前同步的复制节点信息列表。

注意：如果 'auto.create.topics.enable' 在 broker 中配置，一个 topic meta 请求将创建一个 topic， 复制因子和分区数为默认的格式

####### Topic Metadata Request

```
TopicMetadataRequest => [TopicName]
	TopicName => string
```

|字段 | 描述 |
|-----|-----|
| TopicName |  要获取 meta 信息的 topic 名称，如果为空则表示获取所有 topic meta 信息 |

###### Metadata Response

```
MetadataResponse => [Broker][TopicMetadata]
  Broker => NodeId Host Port  (any number of brokers may be returned)
    NodeId => int32
    Host => string
    Port => int32
  TopicMetadata => TopicErrorCode TopicName [PartitionMetadata]
    TopicErrorCode => int16
  PartitionMetadata => PartitionErrorCode PartitionId Leader Replicas Isr
    PartitionErrorCode => int16
    PartitionId => int32
    Leader => int32
    Replicas => [int32]
    Isr => [int32]
```

|字段 | 描述 |
|-----|-----|
| Leader |  对于该分区当前的 leader 节点 ID. 如果当前正在选举 leader 中，则返回 -1 |
| Replicas | 该分区当前作为 leader 的 slaves 节点列表 | 
| Isr | 复制节点列表的子集，leader 的备选节点列表 | 
| Broker | kafka broker 节点的 node id, 主机名，端口等信息 |

###### 可能返回的错误码

- UnknownTopic (3)
- LeaderNotAvailable (5)
- InvalidTopic (17)
- TopicAuthorizationFailed (29)

##### Produce API

生产 API 是用来将消息集合发送到服务端的。为了提升性能允许发送消息集合对于多个 topic 分区在一个单个请求。

生产 API 使用通用的消息集合格式。但是由于没有 offset 被分配在生产者发送的时候，可以使用任何方式填充该值

###### Produce Request

```
v0, v1 (supported in 0.9.0 or later) and v2 (supported in 0.10.0 or later)
ProduceRequest => RequiredAcks Timeout [TopicName [Partition MessageSetSize MessageSet]]
  RequiredAcks => int16
  Timeout => int32
  Partition => int32
  MessageSetSize => int32
```

对于 v1 版或者更高版本的生产请求在请求响应中可以解析出来节流时间

对于 v2 版或者更高版本的生产请求在请求响应中可以解析出来 timestamp 字段

|字段 | 描述 |
|-----|-----|
| RequiredAcks | 该字段用来标识一个请求在响应前应该受到多少个节点确认。如果设置为0 服务端将不会发送任何响应。如果设置为 1， 服务端将等到数据已经写到本地的存储后发送响应。如果设置为 -1 服务端将阻塞等待，直到等所有同步复制节点都写入成功后才会响应 |
| Timeout | 提供一个最大的等待响应的时间，这个 timeout 不能确切的限制请求的时间有如下几个原因：(1) 该超时时间不包括网络延迟时间。(2) 请求处理时才会开启定时器计数，也就是说如果在队列中的多数请求等待的时间是不包括的。(3) 本地写操作是不能中断的，也就是说本地写入时间超过该超时时间是不会起作用的。如果硬限制一个超时时间可以通过在客户端设置 socket 的 timeout 来控制 |
| TopicName | 发送数据对应的 topic 名称 |
| Partition | 发送数据对应的分区号 |
| MessageSetSize | 发送消息集合的大小,即 MessageSet 结构的大小，后面紧随 MessageSet 数据 |
| MessageSet | 一个消息集合，数据结构上面有描述 |

###### Produce Response

```
v0
ProduceResponse => [TopicName [Partition ErrorCode Offset]]
  TopicName => string
  Partition => int32
  ErrorCode => int16
  Offset => int64
 
v1 (supported in 0.9.0 or later)
ProduceResponse => [TopicName [Partition ErrorCode Offset]] ThrottleTime
  TopicName => string
  Partition => int32
  ErrorCode => int16
  Offset => int64
  ThrottleTime => int32
 
v2 (supported in 0.10.0 or later)
ProduceResponse => [TopicName [Partition ErrorCode Offset Timestamp]] ThrottleTime
  TopicName => string
  Partition => int32
  ErrorCode => int16
  Offset => int64
  Timestamp => int64
  ThrottleTime => int32
```

|字段 | 描述 |
|-----|-----|
| Topic | 关联响应的 topic 名称 |
| Partition | 关联响应的分区号 |
| ErrorCode | 该分区的错误。错误是基于分区的，因为给定的分区可能是出于不可信或者其他节点维护状态，其他的请求可能可以成功接受 | 
| Offset | 在该分区的一个消息集合中首条消息会追加 offset 值 |
| Timestamp	 | 如果 topic 使用的是 LogAppendTime ，那么 timestamp 将会被 broker 分配设置到消息集合中。在消息集合中的消息 timestamp 都是一样的, 如果是 CreateTime ，该字段一直返回 -1 , 生产者可以假设 timestamp 是生产请求被接受的时间，如果没有错误返回的情况下. 单位是毫秒 | 
| ThrottleTime | 在该段时间内请求将会被节流处理, 由于影响配额 (如果是 0 则表示请求将不受任何配额影响) | 


###### 可能返回的错误码

TODO

##### Fetch API

The fetch API is used to fetch a chunk of one or more logs for some topic-partitions. Logically one specifies the topics, partitions, and starting offset at which to begin the fetch and gets back a chunk of messages. In general, the return messages will have offsets larger than or equal to the starting offset. However, with compressed messages, it's possible for the returned messages to have offsets smaller than the starting offset. The number of such messages is typically small and the caller is responsible for filtering out those messages.
Fetch requests follow a long poll model so they can be made to block for a period of time if sufficient data is not immediately available.
As an optimization the server is allowed to return a partial message at the end of the message set. Clients should handle this case.
One thing to note is that the fetch API requires specifying the partition to consume from. The question is how should a consumer know what partitions to consume from? In particular how can you balance the partitions over a set of consumers acting as a group so that each consumer gets a subset of partitions. We have done this assignment dynamically using zookeeper for the scala and java client. The downside of this approach is that it requires a fairly fat client and a zookeeper connection. We haven't yet created a Kafka API to allow this functionality to be moved to the server side and accessed more conveniently. A simple consumer client can be implemented by simply requiring that the partitions be specified in config, though this will not allow dynamic reassignment of partitions should that consumer fail. We hope to address this gap in the next major release.


###### Fetch Request

```
FetchRequest => ReplicaId MaxWaitTime MinBytes [TopicName [Partition FetchOffset MaxBytes]]
  ReplicaId => int32
  MaxWaitTime => int32
  MinBytes => int32
  TopicName => string
  Partition => int32
  FetchOffset => int64
  MaxBytes => int32
```

ReplicaId
The replica id indicates the node id of the replica initiating this request. Normal client consumers should always specify this as -1 as they have no node id. Other brokers set this to be their own node id. The value -2 is accepted to allow a non-broker to issue fetch requests as if it were a replica broker for debugging purposes.
MaxWaitTime
The max wait time is the maximum amount of time in milliseconds to block waiting if insufficient data is available at the time the request is issued.
MinBytes
This is the minimum number of bytes of messages that must be available to give a response. If the client sets this to 0 the server will always respond immediately, however if there is no new data since their last request they will just get back empty message sets. If this is set to 1, the server will respond as soon as at least one partition has at least 1 byte of data or the specified timeout occurs. By setting higher values in combination with the timeout the consumer can tune for throughput and trade a little additional latency for reading only large chunks of data (e.g. setting MaxWaitTime to 100 ms and setting MinBytes to 64k would allow the server to wait up to 100ms to try to accumulate 64k of data before responding).
TopicName
The name of the topic.
Partition
The id of the partition the fetch is for.
FetchOffset
The offset to begin this fetch from.
MaxBytes
The maximum bytes to include in the message set for this partition. This helps bound the size of the response.

###### Fetch Response

```
v0
FetchResponse => [TopicName [Partition ErrorCode HighwaterMarkOffset MessageSetSize MessageSet]]
  TopicName => string
  Partition => int32
  ErrorCode => int16
  HighwaterMarkOffset => int64
  MessageSetSize => int32
 
v1 (supported in 0.9.0 or later) and v2 (supported in 0.10.0 or later)
FetchResponse => ThrottleTime [TopicName [Partition ErrorCode HighwaterMarkOffset MessageSetSize MessageSet]]
  ThrottleTime => int32
  TopicName => string
  Partition => int32
  ErrorCode => int16
  HighwaterMarkOffset => int64
  MessageSetSize => int32
```

ThrottleTime	Duration in milliseconds for which the request was throttled due to quota violation. (Zero if the request did not violate any quota.)
TopicName
The name of the topic this response entry is for.
Partition
The id of the partition this response is for.
HighwaterMarkOffset
The offset at the end of the log for this partition. This can be used by the client to determine how many messages behind the end of the log they are.
MessageSetSize
The size in bytes of the message set for this partition
MessageSet
The message data fetched from this partition, in the format described above.


Fetch Response v1 only contains message format v0.
Fetch Response v2 might either contain message format v0 or message format v1.

###### 可能返回的错误码

* OFFSET_OUT_OF_RANGE (1)
* UNKNOWN_TOPIC_OR_PARTITION (3)
* NOT_LEADER_FOR_PARTITION (6)
* REPLICA_NOT_AVAILABLE (9)
* UNKNOWN (-1)



