---
title: 初识Pulsar
date: 2023-11-11 23:28:11
tags: ["MQ", "Pulsar"]
---

![](https://pulsar.apache.org/img/logo-black.svg)
> Apache Pulsar 是 Apache 软件基金会顶级项目，是下一代云原生分布式消息流平台，集消息、存储、轻量化函数式计算为一体，采用计算与存储分离架构设计，支持多租户、持久化存储、多机房跨区域数据复制，具有强一致性、高吞吐、低延时及高可扩展性等流数据存储特性，被看作是云原生时代实时消息流传输、存储和计算最佳解决方案。

#### 主要特性:

* 具备高一致、高可靠、高并发特性
* 采用服务和存储分离架构，支持水平动态扩容
* 支持百万级消息主题
* 非常低的消息发布和端到端的延迟
* 支持多种订阅模式：独占（exclusive）、共享（shared）、灾备（failover）
* 一个 Serverless 的轻量级计算框架 Functions 提供了原生的流数据处理
* 支持多集群，能够无缝的基于地理位置进行跨集群的备份

<!-- more -->
#### 优势

**数据强一致**

Pulsar 版采用 BookKeeper 一致性协议 实现数据强一致性（类似 RAFT 算法），将消息数据备份写到不同物理机上，并且要求是同步刷盘。当某台物理机出故障时，后台数据复制机制能够对数据快速迁移，保证用户数据备份可用。
**高性能低延迟**

Pulsar能够高效支持百万级消息生产和消费，海量消息堆积且消息堆积容量不设上限，支撑了腾讯计费所有场景；性能方面，单集群 QPS 超过10万，同时在时耗方面有保护机制来保证低延迟，帮助您轻松满足业务性能需求。
**百万级 Topic**

Pulsar计算与存储架构的分离设计，使得 Pulsar可以轻松支持百万级消息主题。相比于市场上其他 MQ 产品，整个集群不会因为 Topic 数量增加而导致性能急剧下降。
**丰富的消息类型**

Pulsar提供丰富的消息类型，涵盖普通消息、顺序消息（全局顺序 / 分区顺序）、分布式事务消息、定时消息，满足各种严苛场景下的高级特性需求。
**消费者数量无限制**

不同于 Kafka 的消息消费模式，Pulsar的消费者数量不受限于 Topic 的分区个数，并且会按照一定的算法均衡每个消费者的消息量，业务可按需启动对应的消费者数量。
**多协议接入**

Pulsar的 API 支持 Java、C++、Go 等多语言，并且支持 HTTP 协议，可扩展更多语言的接入，另外还支持开源RocketMQ、RabbitMQ客户端的接入。如果用户只是利用消息队列的基础功能进行消息的生产和消费，可以不用修改代码就完成到 Pulsar的迁移。
**隔离控制**

提供按租户对 Topic 进行隔离的机制，同时可精确管控各个租户的生产和消费速率，保证租户之间互不影响，消息的处理不会出现资源竞争的现象。


#### 物理分区与逻辑分区:
![image](https://main.qcloudimg.com/raw/8fa46c108d316e3cc3bf299b0be7e775.svg)

物理分区：计算与存储耦合，容错需要拷贝物理分区，扩容需要迁移物理分区来达到负载均衡。
逻辑分区：物理“分片”，计算层与存储层隔离，这种结构使得 Apache Pulsar 具备以下优点。
* Broker 和 Bookie 相互独立，方便实现独立的扩展以及独立的容错。
* Broker 无状态，便于快速上、下线，更加适合于云原生场景。
* 分区存储不受限于单个节点存储容量。
* 分区数据分布均匀，单个分区数据量突出不会使整个集群出现木桶效应。
* 存储不足扩容时，能迅速利用新增节点平摊存储负载。


### 消息 ID 生成规则

在 Pulsar 中，每条消息都有自己的 ID（即 MessageID），MessageID 由四部分组成：ledgerId:entryID:partition-index:batch-index。其中：
* partition-index：指分区的编号，在非分区 topic 的时候为 -1。
* batch-index：在非批量消息的时候为 -1。
消息 ID 的生成规则由 Pulsar 的消息存储机制决定，Pulsar 中消息存储原理图如下

![image](https://qcloudimg.tencent-cloud.cn/image/document/5b8f3d165b3977616064fbd95ff003f5.png)

> 如上图所示，在 Pulsar中，一个 Topic 的每一个分区会对应一系列的 ledger，其中只有一个 ledger 处于 open 状态即可写状态，而每个 ledger 只会存储与之对应的分区下的消息。


Pulsar 在存储消息时，会先找到当前分区使用的 ledger ，然后生成当前消息对应的 entry ID，entry ID 在同一个 ledger 内是递增的。每个 ledger 存在的时长或保存的 entry 个数超过阈值后会进行切换，新的消息会存储到同一个 partition 中的下一个 ledger 中。

* 批量生产消息情况下，一个 entry 中可能包含多条消息。
* 非批量生产的情况下，一个 entry 中包含一条消息（producer 端可以配置这个参数，默认是批量的）。

Ledger 只是一个逻辑概念，是数据的一种逻辑组装维度，并没有对应的实体。而 bookie 只会按照 entry 维度进行写入、查找、获取。

#### 分片机制详解：Ledger 和 Entry
> Pulsar 中的消息数据以 ledger 的形式存储在 BookKeeper 集群的 bookie 存储节点上。Ledger 是一个只追加的数据结构，并且只有一个写入器，这个写入器负责多个 bookie 的写入。Ledger 的条目会被复制到多个 bookie 中，同时会写入相关的数据来保证数据的一致性。
 BookKeeper 需要保存的数据包括：
* Journals
    * journals 文件里存储了 BookKeeper 的事务日志，在任何针对 ledger 的更新发生前，都会先将这个更新的描述信息持久化到这个 journal 文件中。
    * BookKeeper 提供有单独的 sync 线程根据当前 journal 文件的大小来作 journal 文件的 rolling。
* EntryLogFile
    * 存储真正数据的文件，来自不同 ledger 的 entry 数据先缓存在内存buffer中，然后批量flush到EntryLogFile中。
    * 默认情况下，所有ledger的数据都是聚合然后顺序写入到同一个EntryLog文件中，避免磁盘随机写。
* Index 文件
    * 所有 Ledger 的 entry 数据都写入相同的 EntryLog 文件中，为了加速数据读取，会作 ledgerId + entryId 到文件 offset 的映射，这个映射会缓存在内存中，称为 IndexCache。
    * IndexCache 容量达到上限时，会被 sync 线程 flush 到磁盘中。
    
三类数据文件的读写交互如下图：

![image](https://qcloudimg.tencent-cloud.cn/image/document/5f9022df4eedd6d45684269fe27050d9.png)

**Entry 数据写入**
1. 数据首先会同时写入 Journal（写入 Journal 的数据会实时落到磁盘）和 Memtable（读写缓存）。
2. 写入 Memtable 之后，对写入请求进行响应。
3. Memtable 写满之后，会 flush 到 Entry Logger 和 Index cache，Entry Logger 中保存数据，Index cache 中保存数据的索引信息，
4. 后台线程将 Entry Logger 和 Index cache 数据落到磁盘。
**Entry 数据读取**
* Tailing read 请求：直接从 Memtable 中读取 Entry。
* Catch-up read（滞后消费）请求：先读取 Index信息，然后索引从 Entry Logger 文件读取 Entry。
**数据一致性保证：LastLogMark**
* 写入的 EntryLog 和 Index 都是先缓存在内存中，再根据一定的条件周期性的 flush 到磁盘，这就造成了从内存到持久化到磁盘的时间间隔，如果在这间隔内 BookKeeper 进程崩溃，在重启后，我们需要根据 journal 文件内容来恢复，这个 LastLogMark 就记录了从 journal 中什么位置开始恢复。
* 它其实是存在内存中，当 IndexCache 被 flush 到磁盘后其值会被更新，LastLogMark 也会周期性持久化到磁盘文件，供 Bookkeeper 进程启动时读取来从 journal 中恢复。
* LastLogMark 一旦被持久化到磁盘，即意味着在其之前的 Index 和 EntryLog 都已经被持久化到了磁盘，那么 journal 在这 LastLogMark 之前的数据都可以被清除了。


#### 消息副本与存储机制

Pulsar 中每个分区 Topic 的消息数据以 ledger 的形式存储在 BookKeeper 集群的 bookie 存储节点上，每个 ledger 包含一组 entry，而 bookie 只会按照 entry 维度进行写入、查找、获取。
> 批量生产消息的情况下，一个 entry 中可能包含多条消息，所以 entry 和消息并不一定是一一对应的。

edger 和 entry 分别对应不同的元数据。
* ledger 的元数据存储在 zk 上。
* entry 除了消息数据部分之外，还包含元数据，entry 的数据存储在 bookie 存储节点上。


![image](https://longerwu-1252728875.cos.ap-guangzhou.myqcloud.com/blogs/pulsar/pulsar-write.png)

每个 ledger 在创建的时候，会在现有的 BookKeeper 集群中的可写状态的 bookie 候选节点列表中，选用 ensemble size 对应个数的 bookie 节点，如果没有足够的候选节点则会抛出 BKNotEnoughBookiesExceptio 异常。选出候选节点后，将这些信息组成 <entry id, ensembles> 元组，存储到 ledger 的元数据里的 ensembles 中。


**消息写入流程**
![image](https://qcloudimg.tencent-cloud.cn/image/document/b3ecbf43626e9bafb76e2ed5b94e580c.png)


客户端在写入消息时，每个 entry 会向 ledger 当前使用的 ensemble 列表中的 Qw 个 bookie 节点发送写入请求，当收到 Qa 个写确认后，即认为当前消息写入存储成功。同时会通过 LAP（lastAddPushed）和 LAC（LastAddConfirmed）分别标识当前推送的位置和已经收到存储确认的位置。
每个正在推送的 entry 中的 LAC 元数据值，为当前时刻创建发送 entry 请求时，已经收到最新的确认位置值。LAC 所在位置及之前的消息对读客户端是可见的。

**消息副本分布**
每个 entry 写入时，会根据当前消息的 entry id 和当前使用的 ensembles 列表的开始 entry id（即key值），计算出在当前 entry 需要使用 ensemble 列表中由哪组 Qw 个 bookie 节点进行写入。之后，broker 会向这些 bookie 节点发送写请求，当收到 Qa 个写确认后，即认为当前消息写入存储成功。这时至少能够保证 Qa 个消息的副本个数。

![image](https://qcloudimg.tencent-cloud.cn/image/document/501d7847c71e3f8fd01aebc06fad6a12.png)

#### 订阅模式
![image](https://qcloudimg.tencent-cloud.cn/image/document/38665c07c06e7ba8dcf96ee32150df37.png)


**独占模式（Exclusive）**

**Exclusive 独占模式（默认模式）**：一个 Subscription 只能与一个 Consumer 关联，只有这个 Consumer 可以接收到 Topic 的全部消息，如果该 Consumer 出现故障了就会停止消费。
Exclusive 订阅模式下，同一个 Subscription 里只有一个 Consumer 能消费 Topic，如果多个 Consumer 订阅则会报错，适用于全局有序消费的场景。

![image](https://qcloudimg.tencent-cloud.cn/image/document/c1f9c9cf1f5d59c18f2f86641de63438.png)




**共享模式（Shared）**
消息通过 round robin 轮询机制（也可以自定义）分发给不同的消费者，并且每个消息仅会被分发给一个消费者。当消费者断开连接，所有被发送给他，但没有被确认的消息将被重新安排，分发给其它存活的消费者。
![image](https://qcloudimg.tencent-cloud.cn/image/document/e40f4037ac99d92f6053c81f4a683e88.png)

**灾备模式（Failover）**
当存在多个 consumer 时，将会按字典顺序排序，第一个 consumer 被初始化为唯一接受消息的消费者。当第一个 consumer 断开时，所有的消息（未被确认和后续进入的）将会被分发给队列中的下一个 consumer。

![image](https://qcloudimg.tencent-cloud.cn/image/document/46ee55c0a3649881282f494e146c3ba5.png)

**KEY 共享模式（Key_Shared）**

当存在多个 Consumer 时，将根据消息的 Key 进行分发，Key 相同的消息只会被分发到同一个消费者。
![image](https://qcloudimg.tencent-cloud.cn/image/document/d0766ee852ec8e8c88451c779c7f8c71.png)

> Key_Shared 本身在使用上存在一定的限制条件，由于其工程实现复杂度较高，在社区版本迭代中，不断有对 Key_Shared 的功能进行改进以及优化，整体稳定性相较 Exclusive,Failover 和 Shared 这三种订阅类型偏弱。如果上述三种订阅类型能满足业务需要，可以优先选用上述三种订阅类型。


**什么时候才考虑用 Key_Shared 订阅模式**

如是普通的生产消费场景，建议直接选用 Shared 模式即可。
若需要让相同 Key 的消息分给同一个消费者，这个时候 Shared 订阅模式无法满足用户需求。有两种方式可以选择：
* 选择 Key_Shared 订阅模式。
* 通过多分区主题 + Failover 订阅模式实现。
 
**什么场景下适合用 Key_Shared 订阅**

Key 数量多且每个 Key 的消息分布相对均匀
消费处理速度快，无消息堆积的情况
如果在生产过程中不能保证上面的两个条件同时满足，建议用 【多分区主题 + Failover 订阅】


多分区主题 + Failover 订阅示例
* 在该模式下，每个分区同时只会分配给一个消费者实例。若消费者数量多于分区数量，超出数量的消费者无法参与消息，可以通过扩容分区数量不小于消费者数量解决。
* 在设计 Key 的时候尽量保证 Key 分布均匀。
* Failover 模式下不支持延时消息。

> Pulsar 内部可以保证单个分区内的消息有序，即如果创建1分区的 Topic 则可以保证全局有序。
单分区的 Topic 会在性能上弱于多分区 Topic，如果希望兼顾性能与有序性， 可以参见 订阅模式 使用 Key-shared 模式进行消费，实现局部有序，标记同一个 key 让需要有序的消息落在同一分区即可。

#### 定时和延时消息
* 定时消息：消息在发送至服务端后，实际业务并不希望消费端马上收到这条消息，而是推迟到某个时间点被消费，这类消息统称为定时消息。
* 延时消息：消息在发送至服务端后，实际业务并不希望消费端马上收到这条消息，而是推迟一段时间后再被消费，这类消息统称为延时消息。
> 实际上，定时消息可以看成是延时消息的一种特殊用法，其实现的最终效果和延时消息是一致的。

```java
String value = "message content";
try {
        //需要先将显式的时间转换为 Timestamp
      long timeStamp = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").parse("2020-11-11 00:00:00").getTime();
      //通过调用 producer 的 deliverAt 方法来实现定时消息
        MessageId msgId = producer.newMessage()
                .value(value.getBytes())
                .deliverAt(timeStamp)
                .send();
} catch (ParseException e) {
        //TODO 添加对 Timestamp 解析失败的处理方法
        e.printStackTrace();
}
```
> 延时消息通过生产者produce的 deliverAfter() 方法实现

使用说明和限制

* 使用定时或延迟消息时，建议与普通消息使用不同的 Topic 来管理，即定时与延迟消息发送到一个固定的 Topic，普通消息发送到另一个 Topic 中，方便后续的管理与维护，增加稳定性。
* 使用定时和延时两种类型的消息时，请确保客户端的机器时钟和服务端的机器时钟（所有地域均为UTC+8 北京时间）保持一致，否则会有时差。
* 定时和延时消息在精度上会有1秒左右的偏差。
* 定时和延时消息不支持 batch 模式（批量发送），batch 模式会引起消息堆积，保险起见，请在创建 producer 的时候把 enableBatch 参数设为 false。
* 定时和延时消息的消费模式仅支持使用 ==Shared 模式==进行消费，否则会失去定时或延时效果（Key-shared 也不支持）。
* 关于定时和延时消息的时间范围，最大均为10天。
使用定时消息时，设置的时刻在当前时刻以后才会有定时效果，否则消息将被立即发送给消费者。
* 设定定时时间后，TTL 的时间依旧会从发送消息的时间点开始算消息的最长保留时间；例如定时到2天后发送，消息最长保留（TTL）如果设置为1天的话，则消息在1天后会被删除，这个时候要确保 TTL 的时间要大于延时的时间，即 TTL 设置成大于等于2天，否则 TTL 到期时，消息会被删除。延时消息同理。

#### 消息重试与死信机制
> 重试 Topic 是一种为了确保消息被正常消费而设计的 Topic 。当某些消息第一次被消费者消费后，没有得到正常的回应，则会进入重试 Topic 中，当重试达到一定次数后，停止重试，投递到死信 Topic 中。
当消息进入到死信队列中，表示 Pulsar已经无法自动处理这批消息，一般这时就需要人为介入来处理这批消息。您可以通过编写专门的客户端来订阅死信 Topic，处理这批之前处理失败的消息。


> 您创建的消费者使用某个订阅名以共享模式订阅了一个 Topic 后，如果开启了 enableRetry 属性，就会自动订阅这个订阅名对应的重试队列。

**主动重试**
当消费者在某个时间没有成功消费某条消息，如果想重新消费到这条消息时，消费者可以发送一条取消确认消息到 Pulsar服务端，Pulsar会将这条消息重新发给消费者。 这种方式重试时不会产生新的消息，所以也不能自定义重试间隔。

```java
while (true) {
    Message msg = consumer.receive();
    try {
        // Do something with the message
        System.out.printf("Message received: %s", new String(msg.getData()));
        // Acknowledge the message so that it can be deleted by the message broker
        consumer.acknowledge(msg);
    } catch (Exception e) {
        // Message failed to process, redeliver later
        consumer.negativeAcknowledge(msg);
    }
}
```


### 消息压缩
#### Pulsar 大消息处理

Pulsar 的消息最大限制默认是 5MB，Producer 发送的消息大小超过5MB会导致消息发送失败。如果客户端发送单条消息大小超过该限制，我们可以采用如下两种方式来处理：
* Chunk Message：Plusar 提供 Chunk Message 功能，开启 Chunk 机制时，客户端能够自动对大消息进行拆分，并保证消息的完整性，在 Consumer 能自动整合消息。
* 压缩消息：主要是对消息数据中相同字符序列进行替换，来压缩消息的大小。Pulsar 支持 LZ4、ZLIB、ZSTD、SNAPPY 四种压缩算法。
我们这里推荐压缩消息对大消息进行处理。


压缩算法分析比较:

* LZ4
LZ4 是一种无损数据压缩算法，可以提供极快的压缩和解压缩速度，对于 CPU 占用少。
* ZLIB
ZLIB 压缩算法是一种常用的无损数据压缩技术，它可以有效地减少收发数据的大小，从而提高网络传输效率和网络容量。ZLIB 压缩算法是基于 Lempel-Ziv 压缩算法的一种变体，可以将原始数据压缩到原来的一半大小以下，并且支持压缩和解压缩操作。
* ZSTD
ZSTD 压缩算法是一种 Huffman 编码的压缩算法，是 LZ77 的一种变种，可以针对不同数据进行有效压缩。它是一种实时编码算法，在处理大数据时可以更快速、更高效地压缩数据。相比其他压缩算法，ZSTD 在提高数据压缩率的同时兼顾压缩速度。
* SNAPPY
SNAPPY 压缩是一种无损压缩技术，它依赖于 LZ77 原理来实现压缩效果。SNAPPY 压缩的核心原理是：只要数据流找到两个字符串之间的重复，就会用一组更短的代码来表示这个字符串，这样就可以减少数据流的大小。

* 吞吐量：LZ4 > SNAPPY > ZSTD > ZLIB
* 压缩比：ZSTD > ZLIB > LZ4 > SNAPPY
* 物理资源方面，SNAPPY 算法占用的网络带宽最多，ZSTD 算法占用的网络带宽最少。

### 生产者
#### 访问模式
消息生成者有多种模式访问 Topic ，可以使用以下几种方式将消息发送到 Topic 。

* Shared：默认情况下，多个生成者可以将消息发送到同一个 Topic。
* Exclusive：在这种模式下，只有一个生产者可以将消息发送到 Topic ，当其他生产者尝试发送消息到这个 Topic 时，会发生错误。只有独占 Topic 的生产者发生宕机时（Network Partition ）该生产者会被驱逐，新的生产者才能产生并向 Topic 发送消息。
* WaitForExclusive：在这种模式下，只有一个生产者可以将消息发送到 Topic 。当已有生成者和 Topic 建立连接时，其他生产者的创建会被挂起而不会产生错误。如果想要采用领导者选举机制来选择消费者的话，可以采用这种模式。

#### 路由模式
当将消息发送到分区 Topic 时，需要指定消息的路由模式，这决定了消息将会被发送到哪个分区 Topic 。 Pulsar 有以下三种消息路由模式，RoundRobinPartition 为默认路由模式。

* RoundRobinPartition：如果消息没有指定 key，为了达到最大吞吐量，生产者会以 round-robin （轮询）方式将消息发布到所有分区。 请注意 round-robin 并不是作用于每条单独的消息，而是作用于延迟处理的批次边界，以确保批处理有效。 如果消息指定了key，分区生产者会根据 key 的 hash 值将该消息分配到对应的分区。这是默认的模式。
* SinglePartition：如果消息没有指定 key，生产者将会随机选择一个分区，并发布所有消息到这个分区。 如果消息指定了key，分区生产者会根据key的hash值将该消息分配到对应的分区。
* CustomPartition：自定义模式，用户可以创建自定义路由模式，通过实现 MessageRouter 接口实现自定义路由规则。